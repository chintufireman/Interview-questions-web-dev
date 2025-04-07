#### Q1 Primary index 
**Answer**: 
1. when u want to save data in file system physically it has 2 conditions
    - ordered
    - key-value
    - unique
2. if these two condition satisfy then we say its primary index

3. eg: when u create a table and it has primary key then by default on that column primary index is applied.

4. u can make primary index either sparse or dense.

5. if u have data in file system then if u want every data for that column should be stored in index data structure then its dense.

6. if u have sorted and unique data for the column u indexed then if u want to store in the form of sparse then it will be stored like this suppose unique id are 1..n and u store it this way in index 1,5,9,13,...n

7. now if u are asked to give me id 3 then it will do binary search and find the key in particular block on logical partition in file system.


#### Q2 Clustered index 
**Answer**: 

1. if the data is ordered and not key-value pair and not unique then it's clustered index.

2. the storing of column values is same as used in primary indexes which are not duplicate.

3. but there might be the case that duplicated values in logical blocks on file system are stored consecutively in 2 different blocks.

4. in this case we will use *block hanker*.

#### Q2 Secondary index 
**Answer**: 

1. As name gives idea already there is already one index present we have to use another index.

2. it is used when ur data is unordered and it has two cases 
    - key
    - non key
3. suppose index on pan number that is not stored in ordered way, which is unique
    - first we create normal index on pan-no column which will be stored in ordered format
    - now this is secondary index first it has index for emid and now u have created index for pan-no column and ordered it.

4. suppose u have to index for name which is not unique and non ordered

    - it will create index name which is not having duplicate value 
    - now there will be a intermediate layer between this index for name and records stored in database.
    - this intermediate layer is called as block of record pointers
    - this block will have multiple duplicate values which will point to original data stored in db. and this whole block will be represented by the index of name u created.
    - now u can see dense and sparse is mixed

#### Q2 what is view in database 
**Answer**: 

1. it is a virtual table which looks like a table but not actually a table.
2. view is a result set of stored query.
3. the query will get compiled and executed and will be stored
4. view is created by 2 ways
    - read only views: not able to do any operation except reading
    - updatable views
5. materialized view: is the updated version of view
    - it simply takes the snapshot of db.
    - suppose u have ur data on server so its copy u will keep on client side.
    - it takes space but comparatively very less space 
6. in view u can bring data from more than one table.

7. advantages of view:
    - to restrict the data access
    - to make complex queries easy
    - to provide data independence: reather than giving full access to user we will give only particular required data's access
    - to present different views on the same data