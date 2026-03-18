Once you've compiled the server core, there are still a number of steps you must go through before being able to start your server.

## 1. Extracting data from the client

The server requires a large amount of data from the client in order to operate. That includes DBC, Map, VMap and MMap files. To extract all that data you need extractors. You can compile them yourself by selecting the _USE_EXTRACTORS_ option when configuring in CMake. Alternatively you can download all the required files from the internet.

#### Windows Guide:
1. **Locate your WoW 1.12.1 client folder**  
   Verify it's build **5875** (bottom-left corner of the login screen).

2. **Copy the extractors and dependencies**  
   Place these files in your World of Warcraft 1.12.1 game folder:  
   - `mapextractor.exe`  
   - `vmapextractor.exe`  
   - `vmap_assembler.exe`  
   - `MoveMapGen.exe`  
   - Any required DLLs from the build output.

3. **Run the extractors by double-clicking (in order)**  
   Open the WoW folder in File Explorer and double-click each one:

   1. Double-click `mapextractor.exe`  
      → This is will extract the dbc and map files.

   2. Double-click `vmapextractor.exe`  
      → This will extract the raw visual map files.

   3. Double-click `vmap_assembler.exe`  
      → This will convert the vmaps into a usable format.

   4. Double-click `MoveMapGen.exe`  
      → This will generate movement maps, so that creatures can navigate all the map geometry properly.
      This is an optional step and it takes a very long time to complete. Be prepared to that it may take between 30 min or hours for it to finish.

      If you choose to skip generating movement maps, mobs will chase their target in a straight line and things like fear movement will not work. If you get an error saying the "mmaps" folder does           not        exist when running the extractor, just create it yourself.

4. **Move extracted folders to your server**  
   Copy from WoW folder: `maps`, `vmaps`, `mmaps` (if created), `dbc`  
   Paste into server data directory (example: `C:\vmangos\data\`)

5. **Organize DBC for build 5875**  
   Rename/move `dbc` → `5875/dbc`  
   Example final path: `C:\vmangos\data\5875\dbc\`
   
   The build number of the 1.12.1 client is 5875.

7. **Cleanup**  
   Delete `Buildings/` and `Cameras/` (if present) from the WoW folder.

#### Linux Guide

1. **Locate your WoW 1.12.1 client folder**  
   Verify it's build **5875** (bottom-left corner of the login screen).

2. **Set your WoW path** (edit once)

   ```bash
   set WOW_PATH '/home/yourname/games/World of Warcraft'
   ```
   Change it to your actual path.

3. **Run extraction (separate commands – run in order)**

   Create the target data directory:
   ```bash
   mkdir -p ~/vmangos/data/5875
   ```
   Go into data folder:
   ```bash
   cd ~/vmangos/data
   ```

   Extract maps and DBC:
   ```bash
   ~/vmangos/bin/Extractors/MapExtractor -i "$WOW_PATH" -o ~/vmangos/data -e 7
   ```

   Move the DBC folder into the version‑specific subdirectory:
   ```bash
   mv ~/vmangos/data/dbc ~/vmangos/data/5875/dbc
   ```

   Extract raw vmap data:
   ```bash
   ~/vmangos/bin/Extractors/VMapExtractor -d "$WOW_PATH/Data"
   ```

   Build final vmaps:
   ```bash
   ~/vmangos/bin/Extractors/VMapAssembler
   ```

   Generate movement maps (adjust thread count automatically):
   ```bash
   ~/vmangos/bin/Extractors/MoveMapGenerator --threads "$(nproc)" --silent --configInputPath ~/vmangos/bin/Extractors/config.json --offMeshInput ~/vmangos/bin/Extractors/offmesh.txt
   ```

   Clean up unused folders:
   ```bash
   rm -rf ~/vmangos/data/{Buildings,Cameras}
   ```

5. **Final structure**

   ```
   ~/vmangos/data/
   ├── 5875/dbc/
   ├── maps/
   ├── vmaps/
   └── mmaps/
   ```

## 3. Setting up the database

The server requires a MySQL database from which to read and save all account, character and world data. Either download the official [MySQL Installer](https://dev.mysql.com/downloads/mysql/5.5.html) or use something like [XAMPP](https://sourceforge.net/projects/xampp/files/XAMPP%20Windows/5.6.12/) which provides a ready to use package of both MySQL and Apache. The recommended versions are either 5.5 or 5.6. Version 5.7 can also work, but some users have reported issues with it.

Once you've installed a MySQL server and created a user with full permissions which the emulator can use, you need to setup the following databases:

- _logs_

This is where log data is stored. To create it use the _logs.sql_ file.

- _realmd_

This is the login server database. Account data is stored here. To create it use the _logon.sql_ file.

- _characters_

This is where all data about characters is stored. To create it use the _characters.sql_ file.

- _mangos_

This is the world database. It contains all game content like items, creatures, texts, etc. To create and populate the world database, you must get the latest world db release from the [database repository](https://github.com/brotalnia/database).

After creating all the databases, you must apply any updates since the last major database release. These updates are referred to as migrations. You will find them in _sql\migrations_ inside the repository. For your convenience, there is a batch script that can be used to merge all migrations into a single file for each database. The world database is updated most often, so you are guaranteed to find an sql file that has to be applied to it. If you fail to apply any updates, the server will print the id of the migrations you are missing and refuse to start.

Once you are done with that, you'll need to define your realms in the database. To do this, go to the `realmd` database and add a new entry inside the `realmlist` table with the address and chosen name of your realm. You can leave all other columns with their default values if you plan on having only one realm. In order to host multiple realms, you'll need to assign a different port for each one. The port in the database entry of the realm must match the port in that realm's config file.

## 4. Editing the config files.

The final step in getting your server running is to make sure everything is correct in the configuration files. There is one for the world server _(mangosd.conf)_, and one for the login server _(realmd.conf)_. Those are simple text files that you can edit in notepad. There are plenty of settings you can change in there, and there is usually a short documentation for each one, but the most important thing you have to check in order to get things running is the MySQL connection string.

It looks like this:
> LoginDatabase.Info              = "127.0.0.1;3306;root;root;realmd"
>
> WorldDatabase.Info              = "127.0.0.1;3306;root;root;mangos"
>
> CharacterDatabase.Info          = "127.0.0.1;3306;root;root;characters"
>
> LogsDatabase.Info               = "127.0.0.1;3306;root;root;logs"

There is a separate connection string for each database. The first part of the string is the IP address of the MySQL server. You can leave that at _127.0.0.1_ if MySQL is running on the same computer as the emulator. The next part is the port, you don't need to touch this. The last 3 values are what you'll need to check. Those are the MySQL user, password, and specific database name. Make sure they are correct.

## 5. Starting your server

Now that everything is configured, make sure MySQL is running, and simply run _realmd.exe_ and _mangosd.exe_ to start your very own vanilla wow server. Once it is done loading you will hear a beep and you may connect to your server.
