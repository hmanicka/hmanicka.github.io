---
layout: post
title: My First Blog - Use Lateral subquery to cut down the Query Run Time
published: false
---

One fine morning, I was asked for a help on a query. At first glance, things seemed very straight forward. I wrote a version of the query with the results expected and boom!It worked. Well, the query fetched the results we wanted, although the time it took to fetch the results were not acceptable, especially in the times we live in. It took about 29 secs...That's definitely not acceptable. So, I started attempting to write other variations of the query and things only went south from there.

Here's the table structure for the tables to be queried:
users
   Column    |           Type           |                   Modifiers                   
-------------+--------------------------+-----------------------------------------------
 uid         | bigint                   | not null default nextval('uid_seq'::regclass)
 username    | text                     | not null
 last_update | timestamp with time zone | 
 
Indexes:
    "users_pskey" PRIMARY KEY, btree (uid)
    "username_key" UNIQUE CONSTRAINT, btree (username)
    "idx_user_id" btree (uid)
    "idx_user_last_update" btree (last_update) CLUSTER
	
 follows
                          Table "public.follows"
    Column    |  Type  |                     Modifiers                     
--------------+--------+---------------------------------------------------
 id           | bigint | not null default nextval('follows_seq'::regclass)
 follower_uid | bigint | not null
 followee_uid | bigint | not null
Indexes:
    "follows_pkey" PRIMARY KEY, btree (id)
    "idx_followee_uid" btree (followee_uid)
    "idx_follower_uid" btree (follower_uid)
	
My first version of the query is as follows, which ran for 29 secs:

{% highlight sql %}
SELECT f.username, f.fl_cnt 
FROM (SELECT followee_uid, username, COUNT(followee_uid) AS fl_cnt FROM follows
INNER JOIN users ON followee_uid = uid AND last_update >= (now() - interval '7 days')
GROUP BY followee_uid, username
HAVING COUNT(followee_uid) > 1000) as f 
ORDER BY f.fl_cnt DESC
LIMIT 100;

I decided to pull the magic card: EXPLAIN comand. So, here's the query plan for the above query:

                                                     QUERY PLAN                                                      
---------------------------------------------------------------------------------------------------------------------
 Limit  (cost=930677.62..930677.67 rows=100 width=20)
   ->  Sort  (cost=930677.62..930678.06 rows=885 width=20)
         Sort Key: (count(follows.followee_uid))
         ->  Hash Join  (cost=930025.92..930670.85 rows=885 width=20)
               Hash Cond: (follows.followee_uid = u.uid)
               ->  HashAggregate  (cost=879259.77..879524.24 rows=75562 width=8)
                     Filter: (count(follows.followee_uid) > 1000)
                     ->  Seq Scan on follows  (cost=0.00..765045.18 rows=76143061 width=8)
               ->  Hash  (cost=50586.19..50586.19 rows=51417 width=20)
                     ->  Index Scan using idx_user_last_update on users u  (cost=0.11..50586.19 rows=51417 width=20)
                           Index Cond: (last_update >= (now() - '7 days'::interval))
(11 rows)
{% endhighlight %}

RESULT: 29 - 32 secs

Attempt 2: Removed the correlated query and used an INNER JOIN

{% highlight sql %}

SELECT u.username, temp.fl_cnt 
FROM users u
INNER JOIN (SELECT f.followee_uid, COUNT(f.followee_uid) as fl_cnt
FROM follows f
GROUP BY f.followee_uid
HAVING COUNT(f.followee_uid) > 1000) as temp ON temp.followee_uid = u.uid
WHERE u.last_update >= (now() - interval '7 days')
ORDER BY f.fl_cnt DESC LIMIT 100;

RESULT: Permance was still sucky.

                                                     QUERY PLAN                                                      
---------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=1028403.19..1034074.81 rows=945270 width=20)
   Filter: (count(f.followee_uid) > 1000)
   ->  Sort  (cost=1028403.19..1028875.82 rows=945270 width=20)
         Sort Key: f.followee_uid, u.username
         ->  Hash Join  (cost=53866.73..995791.73 rows=945270 width=20)
               Hash Cond: (f.followee_uid = u.uid)
               ->  Seq Scan on follows f  (cost=0.00..767269.25 rows=76364417 width=8)
               ->  Hash  (cost=53676.33..53676.33 rows=54401 width=20)
                     ->  Index Scan using idx_user_last_update on users u  (cost=0.11..53676.33 rows=54401 width=20)
                           Index Cond: (last_update >= (now() - '7 days'::interval))
(10 rows)
{% endhighlight %}

As you can see from the above query plans, the Sequential scan on follows table was the bottle neck. And I wanted to address that. Then, I came across Lateral Subqueries and wanted to try it out as I can refer to the outer table in the sub-query section of the query.

Attempt 3:

{% highlight sql %}
SELECT u.username, temp.fl_cnt 
FROM users u,
LATERAL (SELECT COUNT(f.followee_uid) as fl_cnt
FROM follows f
WHERE f.followee_uid = u.uid
HAVING COUNT(f.followee_uid) > 1000) temp
WHERE u.last_update >= (now() - interval '7 days')
ORDER BY f.fl_cnt DESC LIMIT 100

                                               QUERY PLAN                                                
---------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=1967.21..106841265.81 rows=54287 width=20)
   ->  Index Scan using idx_user_last_update on users u  (cost=0.11..53047.21 rows=54287 width=20)
         Index Cond: (last_update >= (now() - '7 days'::interval))
   ->  Aggregate  (cost=1967.10..1967.10 rows=1 width=8)
         Filter: (count(f.followee_uid) > 1000)
         ->  Index Only Scan using idx_followee_uid on follows f  (cost=0.11..1966.08 rows=1011 width=8)
               Index Cond: (followee_uid = u.uid)
(7 rows)
{% endhighlight %}

RESULT: under 2 secs run time

Foot Notes:
Queries tried on Postgres version 9.3.15
