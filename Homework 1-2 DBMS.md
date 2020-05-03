﻿# American Basketball

We chose a database with all American basketball data from 1956 to 2011. 
This is the source of the database: [https://relational.fit.cvut.cz/dataset/BasketballMen](https://relational.fit.cvut.cz/dataset/BasketballMen) 

The database original structure was composed by:
**Size**: 18.3 MB
**Count of tables**: 9
**Count of rows**: 43,787
**Count of columns**: 195

We created other tables, modified some existing tables and deleted all the indexes and constraints except the primary keys.  
This is the final structure of the database: 
#link here of er-diagram~~



## INITIAL UPDATES

    RENAME TABLE `teams` TO `teams_year`;
    
	CREATE TABLE `teams` (
    `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
    `name` varchar(255) NOT NULL DEFAULT '',
    `code` varchar(255) NOT NULL DEFAULT '',
    PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

	INSERT INTO teams (code, name) SELECT DISTINCT ty.franchID, ty.name 
	FROM teams_year ty WHERE ty.year = 
	(SELECT MAX(ty2.year) FROM teams_year ty2 WHERE ty2.franchID = ty.franchID);
    
	CREATE TABLE `league` (
    `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
    `code` varchar(10) NOT NULL DEFAULT '',
    PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

	INSERT INTO league (code) SELECT DISTINCT lgID FROM players_teams;
    
	CREATE TABLE `awards` (
    `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
	`name` varchar(255) NOT NULL DEFAULT '',
	`league_id` int(11) NOT NULL,
	PRIMARY KEY (`id`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8;

	ALTER TABLE `awards_coaches` ADD `league_id` INT NULL DEFAULT NULL AFTER `note`;
    
	ALTER TABLE `awards_players` ADD `league_id` INT NULL DEFAULT NULL AFTER `note`;
    
	UPDATE awards_players ap INNER JOIN league l ON (ap.lgID = l.code) SET ap.league_id = l.id;
    
	UPDATE awards_coaches ac INNER JOIN league l ON (ac.lgID = l.code) SET ac.league_id = l.id;
    
	INSERT INTO awards (name, league_id) SELECT DISTINCT award, league_id FROM awards_players;
    
	INSERT INTO awards (name, league_id) SELECT DISTINCT award, league_id FROM awards_coaches;
    
	ALTER TABLE `awards_coaches` ADD `award_id` INT NOT NULL AFTER `award`;
    
	ALTER TABLE `awards_players` ADD `award_id` INT NOT NULL AFTER `award`;
    
	UPDATE awards_players ap INNER JOIN awards a ON (ap.award = a.name) SET ap.award_id = a.`id`;
    
	UPDATE awards_coaches ac INNER JOIN awards a ON (ac.award = a.name) SET ac.award_id = a.`id`;
    
	ALTER TABLE `awards_players` DROP PRIMARY KEY;
    
	ALTER TABLE `awards_players` ADD PRIMARY KEY (`playerID`,`year`,`award_id`);
    
	ALTER TABLE `awards_coaches` ADD UNIQUE KEY (`coachID`,`year`,`award_id`);
    
	
	ALTER TABLE `awards_players` DROP award ;
    
	ALTER TABLE `awards_coaches` DROP award ;
    
	ALTER TABLE `awards_players` DROP lgID ;
    
	ALTER TABLE `awards_coaches` DROP lgID ;
    
	ALTER TABLE `draft` ADD `league_id` INT NULL DEFAULT NULL;
    
	ALTER TABLE `player_allstar` CHANGE `league_id` `league_code` VARCHAR(255) NULL DEFAULT NULL;
    
	ALTER TABLE `player_allstar` ADD `league_id` INT NULL DEFAULT NULL;
	    
	ALTER TABLE `players_teams` ADD `league_id` INT NULL DEFAULT NULL;
	    
	ALTER TABLE `series_post` ADD `league_id` INT NULL DEFAULT NULL;
	    
	ALTER TABLE `teams_year` ADD `league_id` INT NULL DEFAULT NULL;
	    
	UPDATE draft d INNER JOIN league l ON (d.lgID = l.code) SET d.league_id = l.id;
	    
	UPDATE player_allstar p INNER JOIN league l ON (p.league_code = l.code) SET p.league_id = l.id;
    
	UPDATE players_teams p INNER JOIN league l ON (p.lgID = l.code) SET p.league_id = l.id;
	    
	UPDATE series_post p INNER JOIN league l ON (lgIDWinner = l.code) SET p.league_id = l.id;
	    
	UPDATE teams_year ty INNER JOIN league l ON (ty.lgID = l.code) SET ty.league_id = l.id;

## **QUERIES**

We tried to extrapolate data commonly used to analyze players and teams in the NBA world. 
So we ran 10 very interesting queries in our opinion, trying to touch all the topics explained in the SQL lessons.

#### *1. How many NBA All-Star Game Appearances for each player?*

    SELECT p.useFirst AS name, 
    p.lastName AS last_name, 
    COUNT(pa.season_id) AS total_frequencies 
    FROM players p INNER JOIN player_allstar pa ON (p.playerID = pa.playerID) 
    WHERE pa.league_id = 1 GROUP BY p.playerID ORDER BY total_frequencies DESC;

> **Execution time:** 5.9 ms


#### *2. Top 10 players with highest field goal rates in the new millennium*

    SELECT p.useFirst AS name,
     p.lastName AS last_name, 
     (sum(pt.fgMade)/sum(pt.fgAttempted))*100 AS rate_field_goal 
     FROM players p INNER JOIN players_teams pt ON (p.playerId= pt.playerId) 
     WHERE pt.year>= 2000 and pt.fgMade<= pt.fgAttempted 
     GROUP BY p.playerID 
     HAVING sum(minutes) >1000 ORDER BY  rate_field_goal DESC LIMIT 10;
     
> **Execution time:** 46.6 ms

#### *3. Top 50 players with highest points-per-game in the last year (showing points-per-game, rebounds-per-game, assists-per-game)*

	SELECT  p.useFirst as name, 
	 p.lastName as last_name,
	 ROUND(sum(pt.points)/sum(pt.GP), 2) as pnts_per_game,
	 ROUND(sum(pt.assists)/sum(pt.GP), 2) as assts_per_game, 
	 ROUND(sum(pt.rebounds)/sum(pt.GP), 2) as rbnds_per_game 
	 FROM players_teams pt INNER JOIN players p ON (pt.playerID = p.playerID)
	 WHERE pt.year = (SELECT MAX(year) FROM players_teams) 
	 GROUP BY p.playerID 
	 ORDER BY pnts_per_game DESC LIMIT 50;
> **Execution time:** 27.1 ms

#### *4. Who are the best of the normal ? (same as before, but only among players who have never been selected for the All-Star Game)*

	SELECT  p.useFirst as name,
	 p.lastName as last_name,
	 ROUND(sum(pt.points)/sum(pt.GP), 2) as pnts_per_game,
	 ROUND(sum(pt.assists)/sum(pt.GP), 2) as assts_per_game, ROUND(sum(pt.rebounds)/sum(pt.GP), 2) as rbnds_per_game 
	 FROM players_teams pt 
	 INNER JOIN players p ON (pt.playerID = p.playerID) 
	 WHERE pt.year = (SELECT MAX(year) FROM players_teams) 
	 AND p.playerID NOT IN (SELECT playerID FROM player_allstar WHERE year = pt.year) 
	 GROUP BY p.playerID 
	 ORDER BY pnts_per_game DESC 
	 LIMIT 50;
	 
> **Execution time:** 28 ms

#### *5. Top 10 smallest players to win MVP title ( and how many times they won it )*
	SELECT IFNULL(p.useFirst, p.firstName) AS name, 
	p.lastName AS last_name, 
	p.height AS height_inches, 
	ROUND(p.height*0.0254, 2) AS height_metres, 
	a.name AS award_name, 
	COUNT(1) AS times 
	FROM players p 
	INNER JOIN awards_players ap ON (p.playerID=ap.playerID) 
	INNER JOIN awards a ON (a.id = ap.award_id) 
	WHERE a.name = 'Most Valuable Player' 
	GROUP BY p.playerID 
	ORDER BY p.height ASC 
	LIMIT 10;

> **Execution time:** 1.9 ms

#### *6. Major draft source for the first 3 picks*

    SELECT draftFrom,  
    COUNT(1) AS times 
    FROM draft 
    WHERE draftOverall < 4 
    AND draftFrom IS NOT NULL 
    AND league_id = 1 
    GROUP BY draftFrom 
    ORDER BY times DESC;
    
> **Execution time:** 4.9 ms

#### *7. Coaches who won some awards as coaches but not as players* 

    SELECT DISTINCT p.useFirst AS name, 
    p.lastName 
    FROM players p 
    INNER JOIN players_teams pt ON (p.playerID = pt.playerID)
    WHERE p.playerID IN (SELECT coachID FROM awards_coaches) 
    AND p.playerID NOT IN (SELECT playerID FROM awards_players);
	
> **Execution time:** 84 ms

#### *8. 90s Lakers players with average points per season and number of seasons*

    SELECT p.firstName AS name, 
    p.lastName AS last_name, 
    ROUND(AVG(pt.points),2) AS avg_point, 
    COUNT(1) AS num_seasons 
    FROM players AS p 
    INNER JOIN players_teams AS pt ON (pt.playerID = p.playerID) 
    INNER JOIN teams AS t ON (pt.tmID = t.code) 
    WHERE pt.year BETWEEN 1990 AND 1999 
    AND t.name = 'Los Angeles Lakers' 
    GROUP BY p.playerID, p.firstName, p.lastName;
> **Execution time:** 13.9 ms


#### *9. What teams has Shaq played for?*
	SELECT DISTINCT t.name
	FROM players AS p 
	INNER JOIN players_teams AS pt ON (p.playerID = pt.playerID) 
	INNER JOIN teams AS t ON (pt.tmID = t.code) 
	WHERE p.playerID = pt.playerID 
	AND pt.tmID = t.code 
	AND p.firstName="Shaquille" 
	AND p.lastName = "O'Neal";
> **Execution time:** 179 ms


#### *10. What coaches did Lebron James have?*
	SELECT DISTINCT p_coach.firstName AS name, 
	p_coach.lastName AS last_name 
	FROM players p_coach 
	INNER JOIN coaches c ON (c.coachID = p_coach.playerID) 
	INNER JOIN players_teams pt ON (c.tmID = pt.tmID AND c.year = pt.year) 
	INNER JOIN players p_player ON (pt.playerID = p_player.playerID) 
	WHERE p_player.firstName = 'Lebron' AND p_player.lastName = 'James';

> **Execution time:** 25.4 ms


#FINAL COMMENT HERE

## OPTIMIZATION

#SOMETHING HERE

	ALTER TABLE `players_teams` ADD INDEX `players_teams_players` (`playerID`);
	ALTER TABLE `players_teams` ADD FOREIGN KEY (playerID) REFERENCES players(playerID);
	ALTER TABLE `teams` ADD UNIQUE INDEX `UK_team_code` (`code`);
	ALTER TABLE `players_teams` ADD INDEX `K_year` (`year`);

### FIRST OPTIMIZATION RESULTS

|                     |Before       |After         |Result|
|---------------------|-------------|--------------|---------------|
|Second Query	      |`46.6 ms`    |`30.8 ms`| The time execution decrease is 33.9%
|Third Query          |`27.1 ms`    |`6.3 ms`|The time execution decrease is 76%
|Fourth Query         |`28 ms`    |`5.7 ms`|The time execution decrease is 79.6%
|Ninth Query          |`179 ms`    |`3 ms`|The time execution decrease is 98.3%
|Tenth Query          |`25.4`    |`5.2 ms`|The time execution decrease is 79.5%

