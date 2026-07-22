SELECT DISTINCT m.movie_title FROM movies m JOIN movies_cast mc ON m.movie_id = mc.movie_id WHERE mc.actor_id IN (SELECT actor_id FROM movies_cast GROUP BY actor_id HAVING COUNT(movie_id) >= 2);
