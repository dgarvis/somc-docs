
# Table of Contents

1.  [Overview](#orge94e9c4)
2.  [Key Problems](#org4215768)
3.  [References](#orgdbfe8a3)
4.  [Doing the migration](#org1333094)
    1.  [WARNINGS](#org928914f)
    2.  [Update the Docker Compose](#org1b5fb77)
    3.  [Configure BungeeCord](#org2861e55)
    4.  [Configure BungeeCord to support Bedrock Players](#org2e4e08f)
    5.  [Setup World 1](#org1a8fef2)
    6.  [Setup world 2](#org3c16d27)
    7.  [Add the Simple Portals plugin](#org8d4a8b6)
    8.  [Start the Server and make in world changes](#org6b691a5)
5.  [Adding a Lobby and Using it to Enforce Whitelist](#org7a84216)
6.  [Enabling Backups](#orgf3d03e1)
7.  [Enabling Metric Reporting](#orgab85746)



<a id="orge94e9c4"></a>

# Overview

SoMC scaled up to have two worlds using Multiverse, Multiverse Nether Portals, 
Multiverse Portals, and Multiverse Inventories. This has proven to be unstable and
the goal is to split each world into their own server using BungeeCord as a
proxy between them.


<a id="org4215768"></a>

# Key Problems

-   **Multiverse Inventories was used to separate player inventories:** It appears, that if you keep the save directories named the same and copy over
    the config files and plugin jars for Multiverse Core and Multiverse Inventories
    they will self correct the player inventories after they join.


<a id="orgdbfe8a3"></a>

# References

-   **World 2 Seed:** -2464856654681432429
-   **World 1 Directory:** `world, world_nether, world_the_end`
-   **World 2 Directory:** `Test, Test_nether, Test_the_end`
-   **Original Server Directory:** `data`


<a id="org1333094"></a>

# Doing the migration


<a id="org928914f"></a>

## WARNINGS

This process could be destructive. Make sure the server
has been shutdown and a complete backup created.


<a id="org1b5fb77"></a>

## Update the Docker Compose

1.  Comment out the existing `somc`, `somc-backup`, `somc-exporter` under the ****services****.
2.  In the ****docker-compose.yml**** file, add three new ****volumes****
    
        volumes:
          somc-proxy:
          somc-world1:
          somc-world2:
          [...]
3.  In the ****docker-compose.yml****, under ****services**** add the proxy, and the two new world servers.
    
        services:
          somc-proxy:
            image: itzg/bungeecord
            ports:
              # Main Game
              - "25565:25577"
              # Bedrock
              - "19132:19132/udp"
            volumes:
              - "somc-proxy:/server"
            environment:
              TYPE: BUNGEECORD
            restart: always
          somc-world1:
            image: itzg/minecraft-server
            restart: always
            volumes:
              - "somc-world1:/data"
            environment:
              EULA: "TRUE"
        
              TYPE: "SPIGOT"
              VERSION: "1.20.1"
        
              MEMORY: 10G
              VIEW_DISTANCE: 10
              MAX_PLAYERS: 50
        
              WHITELIST: "WindMagi"
              OPS: "WindMagi"
        
              DIFFICULTY: "normal"
              MOTD: "SoMC: Social and Minecraft"
              ANNOUNCE_PLAYER_ACHIEVEMENTS: "TRUE"
              SNOOPER_ENABLED: "FALSE"
        
              # Note that world 1 name is *world*
              LEVEL: world
          somc-world2:
            image: itzg/minecraft-server
            restart: always
            volumes:
              - "somc-world2:/data"
            environment:
              EULA: "TRUE"
        
              TYPE: "SPIGOT"
              VERSION: "1.20.1"
        
              # We need to ensure the seed value is set
              # to match world 2 from multiverse
              SEED: "-2464856654681432429"
        
              MEMORY: 10G
              VIEW_DISTANCE: 10
              MAX_PLAYERS: 50
        
              WHITELIST: "WindMagi"
              OPS: "WindMagi"
        
              DIFFICULTY: "normal"
              MOTD: "SoMC: Social and Minecraft"
              ANNOUNCE_PLAYER_ACHIEVEMENTS: "TRUE"
              SNOOPER_ENABLED: "FALSE"
        
              # This needs to stay as Test due to multiverse
              # To allow for inventories to get restored correctly.
              LEVEL: Test
4.  Start the instances to verify everything works and create volumes
    
        docker-compose up -d
        docker-compose stop
5.  Create symbolic links to to make accessing the data directories easier. **Note that this assumes that the folder this was created in is called minecraft.**
    
        ln -s /var/lib/docker/volumes/minecraft_somc-proxy/_data/ somc-proxy
        ln -s /var/lib/docker/volumes/minecraft_somc-world1/_data/ somc-world1
        ln -s /var/lib/docker/volumes/minecraft_somc-world2/_data/ somc-world2
6.  Remove data from world directories as we will be restoring to them.
    
        rm -rf somc-world1/*
        rm -rf somc-world2/*


<a id="org2861e55"></a>

## Configure BungeeCord

1.  Copy over the Server Icon
    
        cp data/server-icon.png somc-proxy/
2.  Open the BungeeCord Config File
    
        emacs somc-proxy/config.yml
3.  Under the ****servers**** section remove the default ****lobby****.
4.  Add the two new servers
    
        servers:
          world1:
            motd: World 1
            address: somc-world1:25565
            restricted: false
          world2:
            motd: World 2
            address: somc-world2:25565
            restricted: false
5.  Under the ****priorities**** section remove ****lobby****.
6.  Under the ****priorities**** add our two new servers.
    
        priorities:
        - world1
        - world2
7.  Set the motd to `SoMC: Social Minecraft`
8.  Set ****`ip_forward`**** to ****true****
9.  Save and exit the config file


<a id="org2e4e08f"></a>

## Configure BungeeCord to support Bedrock Players

1.  Download ****Floodgate**** and ****Geyser**** for BungeeCord from [here](https://geysermc.org/download#bungeecord).
2.  Copy both jar files into ****somc-proxy/plugins****


<a id="org1a8fef2"></a>

## Setup World 1

1.  Copy all the data (it is easier to delete, what we don't need)
    
        cp -r data/* somc-world1/
2.  Remove the data for the second world
    
        rm -rf somc-world1/Test
        rm -rf somc-world1/Test_nether
        rm -rf somc-world1/Test_the_end
3.  Remove The Geyser/Floodgate plugins and Multiverse Portals and Netherportals plugins
    
        rm -rf somc-world1/plugins/Geyser*
        rm -rf somc-world1/plugins/floodgate*
        rm -rf somc-world1/plugins/Multiverse-NetherPortals
        rm     somc-world1/plugins/multiverse-netherportals*.jar
        rm -rf somc-world1/plugins/Multiverse-Portals
        rm     somc-world1/plugins/multiverse-portals*.jar
4.  In ****server.properties**** set ****online-mode**** to ****false****.
5.  In ****spigot.yml**** set ****bungeecord**** to ****true****.
6.  In ****bukkit.yml**** set ****connection-throttle**** to ****-1****


<a id="org3c16d27"></a>

## Setup world 2

1.  Copy all the data (it is easier to delete, what we don't need)
    
        cp -r data/* somc-world2/
2.  Move the playerdata, advancements, and datapacks
    
        mv somc-world2/world/playerdata somc-world2/Test/
        mv somc-world2/world/advancements somc-world2/Test/
        mv somc-world2/world/datapacks somc-world2/Test/
3.  Remove the data for the first world
    
        rm -rf somc-world2/world
        rm -rf somc-world2/world_nether
        rm -rf somc-world2/world_the_end
4.  Remove The Geyser/Floodgate plugins and Multiverse Portals and Netherportals plugins
    
        rm -rf somc-world2/plugins/Geyser*
        rm -rf somc-world2/plugins/floodgate*
        rm -rf somc-world2/plugins/Multiverse-NetherPortals
        rm     somc-world2/plugins/multiverse-netherportals*.jar
        rm -rf somc-world2/plugins/Multiverse-Portals
        rm     somc-world2/plugins/multiverse-portals*.jar
5.  In ****server.properties**** set ****online-mode**** to ****false****.
6.  In ****spigot.yml**** set ****bungeecord**** to ****true****.
7.  In ****bukkit.yml**** set ****connection-throttle**** to ****-1****
8.  In ****server.properties**** also change:
    -   level-seed=-2464856654681432429
    -   level-name=Test


<a id="org8d4a8b6"></a>

## Add the Simple Portals plugin

The Simple portal plugin will be used to allow players to move between worlds.

1.  Download [Simple Portals](https://www.spigotmc.org/resources/%E2%8D%9F-simple-portals-%E2%8D%9F-effective-regional-portals-bungeecord-compatible.56772/)
2.  Copy the jar into both worlds plugins directory
    
        cp SimplePortals-*.jar somc-world1/plugins/
        cp SimplePortals-*.jar somc-world2/plugins/


<a id="org6b691a5"></a>

## Start the Server and make in world changes

1.  Start the minecraft servers
    
        docker-compose up -d
2.  Login into the game.
3.  Make the portals to connect worlds
    -   Head to location for portal in game
    -   Place dirt within portal frame at clickable locations
    -   Run the command `/portals sm`
    -   Right-click position one and left-click position two of the portal
    -   Create the portal with `/portals create world1_to_world2` (adjust name as needed)
    -   Link the portals `/portals ss world1_to_world2 world2` (adjust as needed)
4.  Update permissions to allow players to use the portals.
    -   Switch to world 1 with `/server world1`
    -   Open the luck perms editor `/luckperms editor` and click link
    -   Under the default group add ****simpleportals.use**** as ****true****
    -   Switch to world 2 with `/server world2`
    -   repeat steps b and c


<a id="org7a84216"></a>

# Adding a Lobby and Using it to Enforce Whitelist

Create a small lobby server in a super flat / adventure mode / small border
disable whitelist on both sub servers
edit geyser to force login to teh lobby server first
imgrate the old whitelist to this server


<a id="orgf3d03e1"></a>

# Enabling Backups


<a id="orgab85746"></a>

# Enabling Metric Reporting

