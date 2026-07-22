SELECT DISTINCT m.movie_title FROM movies m JOIN movies_cast mc ON m.movie_id = mc.movie_idWHERE mc.actor_id IN (SELECT actor_id FROM movies_cast GROUP BY actor_id HAVINGCOUNT(movie_id) >= 2);
