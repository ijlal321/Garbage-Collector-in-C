This is an example of a Garbage Collector in C. SO you Dont have to manually manage memory.

how this works, 
it has two databases. struct database, and object db.
first application must register all its structs in struct db
and then store the objects (pointing to their corresponsing struct) in object db 

then we use dfs to find nodes which are not reachable. They are leaked memory and must be removed.
we use dfs traversal on nodes. And we use is_visited flag to avoid any loops.\


here is an example

struct 1

struct 2

register struct1
register struct2

obj1
obj2

add to db (obj1)
add to db (obj2)

mark obj1 global

obj1. ptr = obj2

run mld algo

limitations

