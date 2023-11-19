
# Table of Contents

1.  [Overview](#org15e31ee)
2.  [Key Problems](#orgf3a4678)
3.  [References](#org25cf5c0)
4.  [Doing the migration](#org38c81c2)
    1.  [WARNINGS](#org9d970ba)
    2.  [Update the Docker Compose](#org621ad73)
    3.  [Configure BungeeCord](#org592a1be)
    4.  [Configure BungeeCord to support Bedrock Players](#org6b718fe)
    5.  [Setup World 1](#orgef1871b)
    6.  [Setup world 2](#org3008fed)
    7.  [Add the Simple Portals plugin](#orgfe98329)
    8.  [Start the Server and make in world changes](#org8b917db)
5.  [Adding a Lobby and Using it to Enforce Whitelist](#orgc4f654e)
6.  [Enabling Backups](#org527be2b)
7.  [Enabling Metric Reporting](#orgb15bfbf)
8.  [Wrapping Up](#orge8dfada)



<a id="org15e31ee"></a>

# Overview

SoMC scaled up to have two worlds using Multiverse, Multiverse Nether Portals, 
Multiverse Portals, and Multiverse Inventories. This has proven to be unstable and
the goal is to split each world into their own server using BungeeCord as a
proxy between them.


<a id="orgf3a4678"></a>

# Key Problems

-   **Multiverse Inventories was used to separate player inventories:** It appears, that if you keep the save directories named the same and copy over
    the config files and plugin jars for Multiverse Core and Multiverse Inventories
    they will self correct the player inventories after they join.


<a id="org25cf5c0"></a>

# References

-   **World 2 Seed:** -2464856654681432429
-   **World 1 Directory:** `world, world_nether, world_the_end`
-   **World 2 Directory:** `Test, Test_nether, Test_the_end`
-   **Original Server Directory:** `data`


<a id="org38c81c2"></a>

# Doing the migration


<a id="org9d970ba"></a>

## WARNINGS

This process could be destructive. Make sure the server
has been shutdown and a complete backup created.


<a id="org621ad73"></a>

## Update the Docker Compose

1.  Comment out the existing `somc`, `somc-backup`, `somc-exporter` under the **services**.
2.  In the **docker-compose.yml** file, add three new **volumes**
    
        volumes:
          somc-proxy:
          somc-world1:
          somc-world2:
          [...]
3.  In the **docker-compose.yml**, under **services** add the proxy, and the two new world servers.
    
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


<a id="org592a1be"></a>

## Configure BungeeCord

1.  Copy over the Server Icon
    
        cp data/server-icon.png somc-proxy/
2.  Open the BungeeCord Config File
    
        emacs somc-proxy/config.yml
3.  Under the **servers** section remove the default **lobby**.
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
5.  Under the **priorities** section remove **lobby**.
6.  Under the **priorities** add our two new servers.
    
        priorities:
        - world1
        - world2
7.  Set the motd to `SoMC: Social Minecraft`
8.  Set **`ip_forward`** to **true**
9.  Save and exit the config file


<a id="org6b718fe"></a>

## Configure BungeeCord to support Bedrock Players

1.  Download **Floodgate** and **Geyser** for BungeeCord from [here](https://geysermc.org/download#bungeecord).
2.  Copy both jar files into **somc-proxy/plugins**


<a id="orgef1871b"></a>

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
4.  In **server.properties** set **online-mode** to **false**.
5.  In **spigot.yml** set **bungeecord** to **true**.
6.  In **bukkit.yml** set **connection-throttle** to **-1**


<a id="org3008fed"></a>

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
5.  In **server.properties** set **online-mode** to **false**.
6.  In **spigot.yml** set **bungeecord** to **true**.
7.  In **bukkit.yml** set **connection-throttle** to **-1**
8.  In **server.properties** also change:
    -   level-seed=-2464856654681432429
    -   level-name=Test


<a id="orgfe98329"></a>

## Add the Simple Portals plugin

The Simple portal plugin will be used to allow players to move between worlds.

1.  Download [Simple Portals](https://www.spigotmc.org/resources/%E2%8D%9F-simple-portals-%E2%8D%9F-effective-regional-portals-bungeecord-compatible.56772/)
2.  Copy the jar into both worlds plugins directory
    
        cp SimplePortals-*.jar somc-world1/plugins/
        cp SimplePortals-*.jar somc-world2/plugins/


<a id="org8b917db"></a>

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
    -   Under the default group add **simpleportals.portals.** \* as **true**
    -   Switch to world 2 with `/server world2`
    -   repeat steps b and c


<a id="orgc4f654e"></a>

# Adding a Lobby and Using it to Enforce Whitelist

Adding a lobb provides a couple of main benifits:

-   The whitelist can be enforced in on place across the network.
-   Players can change worlds by logging into the game instead of
    having to travel back to spawn of their current world and then
    using a portal to a new world. Allowing them to save their 
    location per world.

To this we are going to add a super flat, migrate the whitelist, and force
entry to the lobby server.

1.  Open **docker-compose.yml** file.
    
        emacs docker-compose.yml
2.  Under the **volumes** section craete a new volume for the lobby
    
        volumes:
          somc-lobby:
          [...]
3.  Under the **serviecs** section add the lobby server.
    
        services: 
          somc-lobby:
            image: itzg/minecraft-server
            restart: always
            volumes:
              - "somc-lobby:/data"
            environment:
              EULA: "TRUE"
        
              TYPE: "SPIGOT"
              VERSION: "1.20.1"
        
              MEMORY: 3G
              VIEW_DISTANCE: 10
              MAX_PLAYERS: 50
        
              WHITELIST: "WindMagi"
              OPS: "WindMagi"
        
              MODE: "adventure"
              DIFFICULTY: "peaceful"
              MAX_WORLD_SIZE: 500
              ALLOW_NETHER: "FALSE"
              FORCE_GAMEMODE: "TRUE"
              GENERATE_STRUCTURES: "FALSE"
              SPAWN_ANIMALS: "FALSE"
              SPAWN_MONSTERS: "FALSE"
              SPAWN_NPCS: "FALSE"
        
              LEVEL_TYPE: FLAT
        
              ONLINE_MODE: "FALSE"
        
              MOTD: "SoMC: Social and Minecraft"
              ANNOUNCE_PLAYER_ACHIEVEMENTS: "FALSE"
              SNOOPER_ENABLED: "FALSE"
        
              LEVEL: world
4.  Create symbolic link to to make accessing the data directory easier.
    
        ln -s /var/lib/docker/volumes/minecraft_somc-lobby/_data/ somc-lobby
5.  Start the new lobby server
    \#+begin-src sh
    docker-compose up -d
    \#+end<sub>src</sub>
6.  In **somc-lobby/spigot.yml** set **bungeecord** to **true**.
7.  In **somc-lobby/bukkit.yml** set **connection-throttle** to **-1**
8.  In **somc-proxy/confgi.yml**, add the new server under the **servers** section
    
        servers:
        lobby:
        motd: Lobby
        address: somc-lobby:25565
        restricted: false
9.  In **somc-proxy/confgi.yml**, **force<sub>default</sub><sub>server</sub>** and set it to **true**
10. In **somc-proxy/confgi.yml**, under **priorities** remove all entries and make it look like:
    
        priorities:
        - lobby
11. Copy the whitelist from the orginal data directory
    
        cp data/whitelist.josn somc-lobby/
12. Install the Plugins Simple Portals, ViaVersion, ViaBackwards, LuckPerms
    
        cp SimplePortals-*.jar somc-lobby/plugins/
        cp data/plugins/Via*.jar somc-lobby/plugins/
        cp data/plugins/LuckPerms*.jar somc-lobby/plugins/
13. Restart the lobby and the proxy
    
        docker-compose restart somc-lobby somc-proxy
14. Login into the game and make sure you are in the lobby world
15. Adjust some gamerulse
    -   doDaylightCycle false
    -   doWeatherCycle false
16. Make the portals to connect each world
    -   Create a portal frame
    -   Place dirt within portal frame at clickable locations
    -   Run the command `/portals sm`
    -   Right-click position one and left-click position two of the portal
    -   Create the portal with `/portals create to_worl1` (adjust name as needed)
    -   Link the portals `/portals ss to_world1 world1` (adjust as needed)
17. Update permissions to allow players to use the portals.
    -   Open the luck perms editor `/luckperms editor` and click link
    -   Under the default group add **`simpleportals.portal.*`** as **true**
18. In the **docker-compose.yml** remove the whitelist value under somc-world1 and somc-world2.
19. Remove the whitelist files
    
        rm somc-world1/whitelist.json
        rm somc-world2/whitelist.json
20. Edit the **server.properties** for each world and set:
    -   white-list = false
    -   enforce-whitelist = false
21. Restart the world servers
    
        docker-compose up -d


<a id="org527be2b"></a>

# Enabling Backups

**Note:** This will assume that you have an ssh config setup to allow for
access to a server called `backup`. We also assume that a folder called ssh
exists with a config and private key that can be used with the backup
dockers.

1.  Ensure the **somc-backup** instance is no longer running and 
    remove it from the docker-compose.yml.
2.  Move the existing borg backup folder into a legacy state
    
        ssh backup mv somc somc-legacy
3.  Create new borg repos for each of the serverrs.
    
        ssh backup mkdir SoMC
        borg init --encryption=repokey backup:SoMC/proxy
        borg init --encryption=repokey backup:SoMC/lobby
        borg init --encryption=repokey backup:SoMC/world1
        borg init --encryption=repokey backup:SoMC/world2
4.  Add 4 new backup services
    
        services:
          somc-proxy-backup:
            image: dmgarvis/minecraft-backup
            restart: always
            environment:
              BORG_REPO: remote_backup:SoMC/proxy
              BORG_PASSPHRASE: (omitted)
              BORG_REMOTE_PATH: borg1
            volumes:
              - somc-proxy:/data:ro
              - ./ssh:/root/.ssh
          somc-lobby-backup:
            image: dmgarvis/minecraft-backup
            restart: always
            environment:
              BORG_REPO: remote_backup:SoMC/lobby
              BORG_PASSPHRASE: (omitted)
              BORG_REMOTE_PATH: borg1
            volumes:
              - somc-lobby:/data:ro
              - ./ssh:/root/.ssh
          somc-world1-backup:
            image: dmgarvis/minecraft-backup
            restart: always
            environment:
              BORG_REPO: remote_backup:SoMC/world1
              BORG_PASSPHRASE: (omitted)
              BORG_REMOTE_PATH: borg1
            volumes:
              - somc-world1:/data:ro
              - ./ssh:/root/.ssh
          somc-world2-backup:
            image: dmgarvis/minecraft-backup
            restart: always
            environment:
              BORG_REPO: remote_backup:SoMC/world2
              BORG_PASSPHRASE: (omitted)
              BORG_REMOTE_PATH: borg1
            volumes:
              - somc-world2:/data:ro
              - ./ssh:/root/.ssh
5.  Now you should be able to start the new backup instances
    \#+bgein<sub>src</sub> sh
    docker-compose up -d
    \#+end<sub>src</sub>


<a id="orgb15bfbf"></a>

# Enabling Metric Reporting

Metric reporting works by having a prometheus exportor for each instance, and
then feeding that into Grafana for visuals.

1.  Open the **docker-compose.yml** file.
2.  Remove the old exporter called **somc-exporter** under services.
3.  Add new exporters under the services for each instance.
    
        services:
          somc-lobby-exporter:
            image: dmgarvis/minecraft_exporter
            restart: always
            environment:
              RCON_HOST: "somc-lobby"
              RCON_PASSWORD: "omitted"
              RCON_PORT: 25575
            volumes:
              - /var/lib/docker/volumes/minecraft_somc-lobby/_data/world:/world:ro
              - /var/lib/docker/volumes/minecraft_somc-lobby/_data/usercache.json:/usercache.json:ro
            depends_on:
              - somc-lobby
          somc-world1-exporter:
            image: dmgarvis/minecraft_exporter
            restart: always
            environment:
              RCON_HOST: "somc-world1"
              RCON_PASSWORD: "omitted"
              RCON_PORT: 25575
            volumes:
              - /var/lib/docker/volumes/minecraft_somc-world1/_data/world:/world:ro
              - /var/lib/docker/volumes/minecraft_somc-world1/_data/usercache.json:/usercache.json:ro
            depends_on:
              - somc-world1
          somc-world2--exporter:
            image: dmgarvis/minecraft_exporter
            restart: always
            environment:
              RCON_HOST: "somc-world2"
              RCON_PASSWORD: "omitted"
              RCON_PORT: 25575
            volumes:
              - /var/lib/docker/volumes/minecraft_somc-world2/_data/world:/world:ro
              - /var/lib/docker/volumes/minecraft_somc-world2/_data/usercache.json:/usercache.json:ro
            depends_on:
              - somc-world2
4.  Save and close the docker-compose file.
5.  Open the **prometheus.yml** file.
6.  Remove the taregt for **somc-exporter**.
7.  Add new exporter targets
    
        - targets:
          - somc-lobby-exporter:8000
        - targets:
          - somc-world1-exporter:8000
        - targets:
          - somc-world2-exporter:8000
8.  Start the new exporter instances
    
        docker-compose up -d
9.  Restart the prometheus instance
    
        docker-compose restart prometheus


<a id="orge8dfada"></a>

# Wrapping Up

A few last things need to be adjusted

-   The SoMC Manager Bot needs to now point to the lobby server for rcon
    requests so that it can control the whitelist.
-   After users have logged into both worlds, this will have corrected their
    inventories and the multiverse core and inventories plugins can be removed.
-   When a user logins again, they need to touch a bed in each world or they
    will spawn at the world center of where ever they die. (problem with multiversue)
-   The metrics reporting api that powers the website will need to have it's 
    queries updated to pull metrics from the new server instances.

