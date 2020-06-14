> :Title color=black
>
> Rooms part 1

> :Author src=github

This week I've started working on the room system for my MUD. My end goal is to generate a small town that players can interacte with.

Building a zone
------------------------
A zone is a grouping of multiple rooms and rooms are connected by a direction. I've decided to represent this as an adjacency list (The directional aspect has yet to be implemented)

```clojure

(defn add-room-if-absent
  "Adds a room to the zone if it doesn't exit and returns the entire zone"
  [zone room]
  (if (contains? zone room)
    zone
    (assoc zone room [])))

(defn add-exit
  "adds an exit from one room to another unidirectionally"
  [zone source-room target-room]
  (-> zone
      (add-room-if-absent target-room)
      (add-room-if-absent source-room)
      (update-in [source-room] #(conj % target-room))
      )
  )
```
We can then generate our zone like so

> :Space space=6px

![Layout](/img/graph-layout.jpg)

```clojure
  (def zone (-> {}
                (add-exit :a :b)
                (add-exit :b :a)
                (add-exit :b :e)
                (add-exit :a :c)
                (add-exit :c :d)
                (add-exit :d :e)))
	=> {:b [:a], :a [:b :c], :c [:d], :d [:e], :e []}
```

Pathfinding
------------
I've also implemented the Dijkstra algorithm so that I can easily find the shortest path to Room X from Room Y

```clojure
(defn build-graph
  "Returns a map for each room in the zone identifying the cost it takes to get there from the source-room"
  [zone source-room]
  (let [rooms (keys zone)]
    (loop [results (assoc-in (zipmap rooms (repeat {:cost ##Inf :prev nil})) [source-room :cost] 0)
           unvisited (set rooms)]
      (if (empty? unvisited)
        results
        (let [current (first (sort-by (comp :cost second) (select-keys results unvisited)))
              current-room (first current)]
          (recur
            (update-costs zone current-room results)
            (disj unvisited current-room))
          )
        )))
  )
  
  (defn update-costs [zone current-room current-costs]
  "returns an updated dijkstra result map by updating the path to all neighbours of the current-room in the zone"
  (let [current-cost (get current-costs :cost)
        neighbours (exits zone current-room)]
    (reduce (fn [x x1]
              (as-> x1 v
                    (get current-costs v)
                    {x1 (if (< (inc current-cost) (get v :cost))
                          (assoc v :cost (inc current-cost) :prev current-room) ;
                          v)
                     }
                    (merge x v))
              )
            current-costs neighbours)
    ))
```

Running the build-graph function returns the following
```clojure
  (build-graph zone :a)
  => {:b {:cost 1, :prev :a}, :a {:cost 0, :prev nil}, :e {:cost 2, :prev :b}, :c {:cost 1, :prev :a}, :d {:cost 2, :prev :c}}
```

Todo
------------
 - We have yet to specify by what direction the rooms are connected by.
 - We're missing the ability to add conditional access to a room.
 - A room is currently represented as a key.