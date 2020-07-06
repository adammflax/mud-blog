> :Title color=black
>
> Rooms part 2

> :Author src=github

It's been a while! I've had to temporary put this project to one side due to time constraints but I'm now back. I'm aiming to make at least one post per week to try and force a habit. At some point
in the future I'll even have nailed down the day I make these posts each week but for now lets take baby steps.

This week I've been working on improvements to the zone system including making the graph weighted, locks and directions

Locks
------
Locks are a mechanism to restrict access to a room unless the player meets a specific requirement. I've stolen the term lock from the paper [Generating_Missions_and_Spaces_for_Adaptable_Play_Experiences](http://sander.landofsand.com/publications/Dormans_Bakkes_-_Generating_Missions_and_Spaces_for_Adaptable_Play_Experiences.pdf)
which I intended to implement so that I can procedurally generate dungeons using L-Systems. Currently I have two different kind of locks based on if the player has an item or a flag.

To kick things off I've had to design the rough layout of the player structure:

```clojure
(def player {
             :name      "wibble"
             :class     :mage
             :ability   {:name :resilient :description "Takes less damage when health is below 30%"}
             :stats     {:health              100
                         :critical-strike     5
                         :critical-multiplier 1.5
                         :attack-speed        0.8
                         :armour              15
                         :resistance          15
                         :damage              10
                         :magic-damage        10
                         :inventory-slots     30}
             :flags     []
             :inventory [`(:rope 1) `(:healing-potion 10) `(:gold 1000)]
             })
```
I've made each item in the inventory be a tuple so that I can support stackable items. Flags are truthy values associated with the player. For instance if the player
decides to choose chaotic-good as an alignment I would add that as a value to the flags vector.

To help implement locks in my game I've added in two new helper functions.

```clojure
(defn has-item?
  "returns true if the player has the item in the inventory otherwise false"
  [item player]
  (true? (some #(= (first %) item) (player :inventory)))
  )

(defn has-flag?
  "returns true if the player has the specified flag set otherwise false"
  [flag player]
  (true? (some #(= % flag) (player :flags)))
  )
```
I've used the ```true?``` function because ```some``` will either return true or nil which I think is a gotcha. 


Redefining the vertex
----------------------
The verticies now need to store the direction, weight and the lock so I've needed to update the ```add-exit``` function
```clojure
(defn add-exit
  "adds an exit from one room to another unidirectional"
  ([zone source-room target-room direction cost] (add-exit zone source-room target-room direction cost []))
  ([zone source-room target-room direction cost locks]
   (-> zone                                                 ;first lets ensure that zone has both rooms
       (add-room-if-absent target-room)
       (add-room-if-absent source-room)
       (update-in [source-room] #(conj % {:target target-room :direction direction :cost cost :locks locks})) ;append to source's adjacency vector
       )
   ))
```

We can now build the following zone with the following code

![Layout](/img/graph-layout2.jpg)

```clojure
  (def zone (-> {}
                (add-exit :a :b :south 1)
                (add-exit :b :a :north 1)
                (add-exit :b :e :south-east 4)
                (add-exit :a :c :east 2 [(partial has-item? :candle)])
                (add-exit :c :d :south 1)
                (add-exit :d :e :south 1)))
				
   => {
		:b [{:target :a, :direction :north, :cost 1, :locks []} {:target :e, :direction :south-east, :cost 4, :locks []}],
		:a [{:target :b, :direction :south, :cost 1, :locks []}
			{:target :c, :direction :east,  :cost 2, :locks [#someFnAddress]}
		   ],
		:e [],
		:c [{:target :d, :direction :south, :cost 1, :locks []}],
		:d [{:target :e, :direction :south, :cost 1, :locks []}]
	  }
```

Updating Dijkstra
------------
Updating Dijkstra to take into account weights and locks was trivial we just had to make some minor adjustments to the calculation of costs and exits

```clojure
(defn exits
  "Lists the exits of a room"
  [zone player room]
  (map :target (filter #(has-access? player (% :locks)) (room-edges zone room))))

(defn build-graph
  [zone player source-room]
  (let [rooms (keys zone)]
    (loop [results (assoc-in (zipmap rooms (repeat {:cost ##Inf :prev nil})) [source-room :cost] 0)
           unvisited (set rooms)]
      (if (empty? unvisited)
        results
        (let [current (first (sort-by (comp :cost second) (select-keys results unvisited)))
              current-room (first current)
              current-cost (get (second current) :cost)
              neighbours (exits zone player current-room)
              calculate-new-cost (fn [zone current-room target-room] (+ current-cost ((room-edges zone current-room target-room) :cost)))]
          (recur
            (reduce (fn [x x1]
                      (as-> x1 v
                            (results v)
                            {x1 (if (< (calculate-new-cost zone current-room x1) (v :cost))
                                  (assoc v :cost (calculate-new-cost zone current-room x1) :prev current-room) v)}
                            (merge x v))
                      )
                    results neighbours)
            (disj unvisited current-room))
          )
        )))
  )
```
running this code gives us the following output (the player doesn't have a candle item)
```clojure
(build-graph zone player :a)
=>
{:b {:cost 1, :prev :a},
 :a {:cost 0, :prev nil},
 :e {:cost 5, :prev :b},
 :c {:cost ##Inf, :prev nil},
 :d {:cost ##Inf, :prev nil}}
```

Next Steps
------------
This week I stumbled upon this article [Building an E-Commerce Marketplace Middleware in Clojure](https://www.demystifyfp.com/clojure/marketplace-middleware/intro/) so I will look at getting clojure to load my room data from a database next.