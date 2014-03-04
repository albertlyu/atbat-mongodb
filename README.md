atbat-mongodb
=============

## The Gist
This is a Perl project that pulls game, at-bat and pitch data from MLB's AtBat servers and shoves them into a local Mongo Database. 

When you first get setup you can pull an entire year or month of data. From then on, each time you
run the program it will pickup where it left off, keeping your database up-to-date with the baseball season.

Disclaimer: You'll probably need some software development background to get up and running with this project. At the least a comfort
with databases.

---

## Prerequisites

### Perl
You'll need to install Perl and a few external modules from CPAN. Getting Perl will be different for all of the Operating Systems
so I won't go into it here, but I'll list a few notes..

* *Windows*: Google ActiveState or StrawberryPerl
* *MacOS*: You already have Perl. Thanks Apple! You'll need the Developer Tools installed to install all of the modules required below. Search
the App Store for the Developer Tools.
* *Linux*: You know what you're doing. Continue.

#### Perl Modules Required
You'll need to install these modules if you don't have them installed already

* Config::Properties
* Log::Log4perl
* File::Basename
* Getopt::Long
* LWP
* XML::Simple
* Data::Dumper
* Date::Parse
* DateTime
* Storable
* MongoDB

Normally you would use cpan to install each module. Something like...

    $ sudo cpan install Config::Properties
    
Or if you're on MacOS you may need to run it through Perl like...

    $ perl -MCPAN -e 'install Config::Properties'
  

### MongoDB
You need a MongoDB installation. 

http://www.mongodb.org

You don't need to configure anything, just install Mongo and start the mongod process.

---

## Your First Run
When you're first getting setup and your database is empty, you'll first need to sync a specific day, month or year. 
I suggest you sync the current month, which takes about 5-10 minutes depending on your Internet connection.

    ./atbatETL.pl --year=2013 --month=06

If you're on a fast pipe, you might as well just do a full year. I can grab a year in about 40 minutes. Running
the program like this will grab an entire year.

    ./atbatETL.pl --year=2013

Note that the program logs quite a bit of interesting output to the log filename listed in the *log4perl.conf* file. By default this is a file
called *mlbatbat.log*. I suggest you tail the log file and watch the days and games roll by. A snippet of the output is...

    2013/06/29 15:42:53 DEBUG [Kruser.MLB.AtBat] Getting game roster details from http://gd2.mlb.com/components/game/mlb/year_2013/month_06/day_28/gid_2013_06_28_slnmlb_oakmlb_1/players.xml
    2013/06/29 15:42:53 DEBUG [Kruser.MLB.AtBat] Getting at-bat details from http://gd2.mlb.com/components/game/mlb/year_2013/month_06/day_28/gid_2013_06_28_chnmlb_seamlb_1/inning/inning_all.xml
    2013/06/29 15:42:54 DEBUG [Kruser.MLB.Storage.Mongo] Saved 80 at bats to the 'atbats' collection
    2013/06/29 15:42:55 DEBUG [Kruser.MLB.Storage.Mongo] Saved 287 pitches to the 'pitches' collection
    2013/06/29 15:42:55 DEBUG [Kruser.MLB.AtBat] Getting game roster details from http://gd2.mlb.com/components/game/mlb/year_2013/month_06/day_28/gid_2013_06_28_chnmlb_seamlb_1/players.xml
    2013/06/29 15:42:55 DEBUG [Kruser.MLB.AtBat] Getting at-bat details from http://gd2.mlb.com/components/game/mlb/year_2013/month_06/day_28/gid_2013_06_28_phimlb_lanmlb_1/inning/inning_all.xml
    2013/06/29 15:42:58 DEBUG [Kruser.MLB.Storage.Mongo] Saved 88 at bats to the 'atbats' collection
    2013/06/29 15:42:59 DEBUG [Kruser.MLB.Storage.Mongo] Saved 332 pitches to the 'pitches' collection
    2013/06/29 15:42:59 DEBUG [Kruser.MLB.AtBat] Getting game roster details from http://gd2.mlb.com/components/game/mlb/year_2013/month_06/day_28/gid_2013_06_28_phimlb_lanmlb_1/players.xml
    2013/06/29 15:43:00 INFO [Kruser.MLB.AtBat] Finished retrieving data for 2013-06-28.
    2013/06/29 15:43:00 INFO [Kruser.MLB.AtBat] The target date for 2013-06-29 is today, in the future, or late last night. Exiting soon....
    2013/06/29 15:43:02 DEBUG [Kruser.MLB.Storage.Mongo] Saved 62 players to the 'players' collection

Once your initial run finishes, the next time you run it without args it will pickup where it left off. I suggest running it on a cron or 
scheduled task for noon eastern time daily. I won't let it read before 8AM as a precaution against crazy rain-out days.

    ./atbatETL.pl
    
---

## Your New Database!!
Startup the *mongo* shell program found in your installs bin directory.

    RYANs-MacBook-Pro:dsire kruser$ /Applications/mongodb-osx-x86_64-2.2.0/bin/mongo
    MongoDB shell version: 2.2.0
    connecting to: test
    > 

### Collections
Collections in MongoDB are analygous to tables in a relational database. You'll have five of them which you can see from the *show collections* 
command below. Note that when you first open the mongo shell you'll need to switch the context to the *mlbatbat* database using the *use mlbatbat*
command as you see below.

    > use mlbatbat
    switched to db mlbatbat
    > show collections
    atbats
    games
    pitches
    players
    system.indexes
    > 

You should have lots of data in your four collections as you can see below using the *count()* function. If you don't see lots of records then
start over at the beginning as something went wrong with the data collection.

    > db.games.count()
    1222
    > db.players.count()
    1166
    > db.atbats.count()
    90444
    > db.pitches.count()
    346822
    > 

### Indexes
Note that I haven't created indexes on any of your database collections by default. You may wish to place these on your index
depending on the type of research you're doing. Of course, this is all optional, but it would provide performance boosts if you're
doing a lot of queries.

Read up on MongoDB indexes for more information.
http://docs.mongodb.org/manual/core/indexes/

For the http://PitchFX.org site I have started with these indexes. You can't go wrong with these if you don't care about the slight storage overhead.
    
    db.players.ensureIndex({'first':1,'last':1});
    db.pitches.ensureIndex({'atbat.pitcher':1,'tfs_zulu':1});
    db.pitches.ensureIndex({'atbat.pitcher':1});
    db.pitches.ensureIndex({'atbat.batter':1,'tfs_zulu':1});
    db.pitches.ensureIndex({'atbat.batter':1});
    db.pitches.ensureIndex({'atbat.p_throws':1});
    db.pitches.ensureIndex({'atbat.stand':1});
    db.pitches.ensureIndex({'atbat.o_start':1});
    db.pitches.ensureIndex({'game.game_type':1});
    db.pitches.ensureIndex({'inning.number':1});
    db.pitches.ensureIndex({'on_1b':1});
    db.pitches.ensureIndex({'on_2b':1});
    db.pitches.ensureIndex({'on_3b':1});
    db.pitches.ensureIndex({'tfs_zulu':1});
    db.atbats.ensureIndex({'pitcher':1,'start_tfs_zulu':1});
    db.atbats.ensureIndex({'batter':1,'start_tfs_zulu':1});
    db.atbats.ensureIndex({'pitcher':1});
    db.atbats.ensureIndex({'batter':1});
    db.atbats.ensureIndex({'p_throws':1});
    db.atbats.ensureIndex({'stand':1});
    db.atbats.ensureIndex({'o_start':1});
    db.atbats.ensureIndex({'game.game_type':1});
    db.atbats.ensureIndex({'start_tfs_zulu':1});
    db.atbats.ensureIndex({'inning.number':1});
    db.atbats.ensureIndex({'pitch.on_1b':1});
    db.atbats.ensureIndex({'pitch.on_2b':1});
    db.atbats.ensureIndex({'pitch.on_3b':1});

### Some sample functions
I won't have a lot of information here. This part is mostly up to you, but I want to give you some foo to get you excited.

#### How many 100+ MPH pitches were thrown in May 2013? How many were thrown for balls and how many for strikes?
To find this data we'll query the *pitches* collection. Note that we're specifying the months in an 
array of 0-11 instead of 1-12. So 3=April, 4=May, etc.

    > db.pitches.find({"start_speed":{$gte:100}, "tfs_zulu":{$gte:new Date(2013,4,1), $lt:new Date(2013,5,1)}}).count();
    42

We see that there were *42* total in the month of May 2013. Let's split them up and see how many were thrown for strikes, how many were balls
and how many were hit into play. To do this, we'll use a *group()* function instead of a *find()*.

    > db.pitches.group (
    {
       key: {"type": true}, 
       cond: {"start_speed":{$gte:100}, "tfs_zulu":{$gte:new Date(2013,4,1), $lt:new Date(2013,5,1)}},
       initial: {sum: 0}, 
       reduce: function(doc, prev) { prev.sum += 1}
    });
    
The results of the query above are...

    [
	    {
		    "type" : "B",
		    "sum" : 15
	    },
	    {
		    "type" : "X",
		    "sum" : 9
	    },
	    {
		    "type" : "S",
		    "sum" : 18
	    }
    ]
    
By using *group()* we can see the breakdown of the league's 100+MPH pitches
* 15 balls (B)
* 18 strikes (S)
* 9 hit into play (X)

#### What is Joe Mauer's Batting Average with 2 strikes in all of 2013?
First we'll need to find Joe Mauer's AtBat ID.

    > db.players.find({'last':'Mauer'}).pretty();
    {
	    "_id" : ObjectId("51ceff10d0930a21010016ad"),
	    "first" : "Joe",
	    "last" : "Mauer",
	    "id" : NumberLong(408045)
    }
    > 

Now that we know his ID is *408045*, we can query the *atbats* collection for the data we need. Notice that I preserved the *id* property
from the MLB data and didn't try to fit that in the MongoDB *_id* field.

We'll run two queries, one for total at-bats with two strikes and one for total hits.

    > db.atbats.find({"batter":408045,"start_tfs_zulu":{$gte:new Date(2013,0,1), $lt:new Date(2014,0,1)}, "s":{$gte:2}, "event":/Single|Double|Triple|Home Run/}).count();
    54
    > db.atbats.find({"batter":408045,"start_tfs_zulu":{$gte:new Date(2013,0,1), $lt:new Date(2014,0,1)}, "s":{$gte:2}, "event":{$not:/Walk|Sacrifice/}}).count();
    183

The queries tell us that Joe Mauer is 54 for 183, or *.295* in 2013 when he has two strikes. Notice that we used *$gte:2* since the at-bat
will be reported to have three strikes when the batter strikes out, and we certainly want to include that.

The example above would have been much more performant with a MongoDB aggregate $match and a $group that aggregated 
the at-bats and hits together. I kept this as two queries for simplicity. For more information on MongoDB aggregation, 
go here http://docs.mongodb.org/manual/reference/aggregation/


---

## Why MongoDB?
MongoDB is a document based "nosql" database. Baseball data is particularly relational, but I was interested in seeing if
we could make it a little less so and take advantage of the speed of MongoDB. When I say "speed" I'm speaking of the speed
of both development and usage. You see, I've defined no schema. Instead, I've pretty much taken the XML documents from 
the At-Bat servers, sucked them into a POPO (plain old Perl object), and fed them into Mongo. It was simple and FUN! 

Now I did shuffle some data around, making sure a pitch document contained enough information about the at-bat and game to be useful and the same
for at-bats, but for the most part the data stayed with the property names that you find in the MLB At-Bat documents.

Additionally MongoDB has built-in support for cloud scaling and map-reduce functions. Unlike MySQL, SQLServer, etc., we can run Javascript functions
in the Mongo shell, and even in a map-reduce setup.

---
## Contribute
Fork my repo, please! I accept pull requests so let's chat if you're interested in contributing.

---
## Future
### Speed
MongoDB is fast on inserts, 99% of the time in running this program is spent waiting for HTTP GET requests to return from the mlb servers.
I would like to put the *_save_game_data* method in AtBat.pm into a thread pool. Originally I had it this way but Perl's LWP is a little
flaky across threads and I didn't want to spend too much time on the issue. If we were able to startup each *_save_game_data* in a thread it would
cut down the runtime of the program to 10% or less. That said, once the initial sync is in a place you like it, you simply run it without args
on a cron/daily schedule and you'll maintain an up-to-date database and you don't really care about runtime speed, only database speed.

### Python?
I think Python might have been a wiser choice than Perl for this project, but I can slap Perl together a little faster so I went with that. 
I'm thinking a port to Python would be great, provided I'm able to give into the whitespace rules of the language. So maybe I'll do that
soon, maybe not.

### ElasticSearch Storage
I would like to have other storage options in addition to MongoDB. I would especially like to see an ElasticSearch.pm module in *Storage*.
ElasticSearch offers some faceting capabilities that would give us extra quick looks without the overhead of the mongo group function. Before starting
an ElasticSearch option though I think it would be wise to look at using a Mongo River that stores to ElasticSearch downstream of Mongo.

### MongoDB Options
Right now the program only connects to mongod running on the localhost, default port, without credentials. If this were a commercial product,
this would be quite rediculous. As it stands, I don't need more than that. But yes, eventually I'd like to support running against a remote
MongoDB instance.
