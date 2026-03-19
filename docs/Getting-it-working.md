Once you've compiled the server core, there are still a number of steps you must go through before being able to start your server.

## 1. Extracting data from the client

The server requires a large amount of data from the client in order to operate. That includes DBC, Map, VMap and MMap files. To extract all that data you need extractors. You can compile them yourself by selecting the _USE_EXTRACTORS_ option when configuring in CMake. Alternatively you can download all the required files from the internet.

#### Extracting data from client on Windows:
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

#### Extracting data from client on Linux
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

#### Setting up database on Windows
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

#### Setting up database on Linux

To set up the database on Linux, you can either import the full database that already includes the latest migrations, or import the latest database release first and then apply the most recent migrations separately.

Further down the page is steps to do both.

#### Set username and password for DB, so it can be reused in the following steps:
```bash
read -p "Database user [mangos]: " DB_USER; DB_USER=${DB_USER:-mangos}
read -sp "Database password [mangos]: " DB_PASS; echo; DB_PASS=${DB_PASS:-mangos}
```
Default username and password is mangos.

#### Set up mariaDB databases and user
```bash
mariadb -u root -p -e "CREATE DATABASE IF NOT EXISTS realmd DEFAULT CHARSET utf8 COLLATE utf8_general_ci; CREATE DATABASE IF NOT EXISTS mangos DEFAULT CHARSET utf8 COLLATE utf8_general_ci; CREATE DATABASE IF NOT EXISTS characters DEFAULT CHARSET utf8 COLLATE utf8_general_ci; CREATE DATABASE IF NOT EXISTS logs DEFAULT CHARSET utf8 COLLATE utf8_general_ci; CREATE USER IF NOT EXISTS '${DB_USER}'@'localhost' IDENTIFIED BY '${DB_PASS}'; GRANT ALL PRIVILEGES ON realmd.* TO '${DB_USER}'@'localhost'; GRANT ALL PRIVILEGES ON mangos.* TO '${DB_USER}'@'localhost'; GRANT ALL PRIVILEGES ON characters.* TO '${DB_USER}'@'localhost'; GRANT ALL PRIVILEGES ON logs.* TO '${DB_USER}'@'localhost'; FLUSH PRIVILEGES;"
```
*You will be prompted for the mariaDB root password.*

Start with entering a bash session, to ensure commands will still run in your shell.
Enter bash session by typing:
```bash
bash
```
If you at any point want to exit the bash session, type:
```bash
exit
```

For the below steps you'll need `p7zip` installed. If not present:
* **Debian/Ubuntu:** `sudo apt install p7zip-full`
* **Red Hat/Fedora:** `sudo dnf install p7zip`
* **Arch:** `sudo pacman -S p7zip`

If you don't want to install p7zip or have another tool to extract 7z or zip files, just cd into /tmp/ and extract it into /tmp/ 

#### Importing the latest database with latest migrations pre-applied (fast DB setup):
```bash
ZIP_URL=$(curl -s https://api.github.com/repos/vmangos/core/releases/tags/db_latest | grep -o '"name": "db-[0-9a-f]*\.zip"' | sed 's/"name": "\([^"]*\)"/\1/' | head -n1 | sed 's|^|https://github.com/vmangos/core/releases/download/db_latest/|') && wget -qO /tmp/db-latest.zip "$ZIP_URL" && 7z x /tmp/db-latest.zip -o/tmp/ -y && mariadb -u"${DB_USER}" -p"${DB_PASS}" --max-allowed-packet=256M realmd < /tmp/db_dump/logon.sql && mariadb -u"${DB_USER}" -p"${DB_PASS}" --max-allowed-packet=256M characters < /tmp/db_dump/characters.sql && mariadb -u"${DB_USER}" -p"${DB_PASS}" --max-allowed-packet=256M mangos < /tmp/db_dump/mangos.sql && mariadb -u"${DB_USER}" -p"${DB_PASS}" --max-allowed-packet=256M logs < /tmp/db_dump/logs.sql
```
You have now setup the DB and can proceed to step 4. "Editing the config files".

#### Importing the latest database release and importing migrations separately (if you did the above you can skip this):

#### 1. **Get the latest database file and download it**
This command fetches the repository page, extracts all `.7z` filenames, sorts them to find the newest one (by date in filename), and downloads it.
```bash
LATEST_DB=$(curl -s https://api.github.com/repos/brotalnia/database/contents/ | grep -o '"name": "world_full_[^"]*\.7z"' | sed 's/"name": "\(.*\)"/\1/' | while read f; do date_part=$(echo "$f" | sed 's/world_full_\(.*\)\.7z/\1/'); echo "$(date -d "$(echo "$date_part" | sed 's/_/ /g')" +%s 2>/dev/null) $f"; done | sort -n | tail -n1 | cut -d' ' -f2-) && wget -qO "/tmp/$LATEST_DB" "https://github.com/brotalnia/database/raw/master/$LATEST_DB"
```

#### 2. **Extract the database**

```bash
7z x "/tmp/$LATEST_DB" -o/tmp/ -y
```
This extracts the `.sql` file to `/tmp/`.

#### 3. **Import the base world database**
```bash
mariadb -u"${DB_USER}" -p"${DB_PASS}" mangos < "/tmp/$(basename "$LATEST_DB" .7z).sql"
```

#### 4. **Run the vmangos migration merge script**
Navigate to the core migrations folder and run the provided script to combine all updates.
```bash
cd ~/vmangos/core/sql/migrations && ./merge.sh
```
This generates combined update files like `world_db_updates.sql`, `logs_db_updates.sql`, etc., in the same directory.

#### 4. **Import logs,logon and character SQL files**
```bash
mariadb -u"${DB_USER}" -p"${DB_PASS}" logs < ~/vmangos/core/sql/logs.sql
```
```bash
mariadb -u"${DB_USER}" -p"${DB_PASS}" realmd < ~/vmangos/core/sql/logon.sql
```
```bash
mariadb -u"${DB_USER}" -p"${DB_PASS}" characters < ~/vmangos/core/sql/characters.sql
```

#### 5. **Import the merged migration files**
Import the migrations SQL files one by one:
```bash
mariadb -u"${DB_USER}" -p"${DB_PASS}" mangos < ~/vmangos/core/sql/migrations/world_db_updates.sql
```
```bash
mariadb -u"${DB_USER}" -p"${DB_PASS}" logs < ~/vmangos/core/sql/migrations/logs_db_updates.sql
```
```bash
mariadb -u"${DB_USER}" -p"${DB_PASS}" realmd < ~/vmangos/core/sql/migrations/logon_db_updates.sql
```
```bash
mariadb -u"${DB_USER}" -p"${DB_PASS}" characters < ~/vmangos/core/sql/migrations/characters_db_updates.sql
```

---

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


### Linux copy and configure server config files
```bash
mkdir ~/vmangos/logs
```
```bash
cp ~/vmangos/etc/mangosd.conf.dist ~/vmangos/etc/mangosd.conf
```
```bash
cp ~/vmangos/etc/realmd.conf.dist ~/vmangos/etc/realmd.conf
```
```bash
sed -i "s|^DataDir.*|DataDir = \"../data\"|" ~/vmangos/etc/mangosd.conf
```
```bash
sed -i "s|^LogsDir.*|LogsDir = \"../vmangos/logs\"|" ~/vmangos/etc/mangosd.conf
```
```bash
sed -i "s|^LogsDir.*|LogsDir = \"../vmangos/logs\"|" ~/vmangos/etc/realmd.conf
```

> **Important:** Open `~/vmangos/etc/mangosd.conf` and `~/vmangos/etc/realmd.conf` in a text editor and verify the database connection strings match your `${DB_USER}` and `${DB_PASS}`. They should look like:
> ```
> LoginDatabase.Info    = "127.0.0.1;3306;mangos;mangos;realmd"
> WorldDatabase.Info    = "127.0.0.1;3306;mangos;mangos;mangos"
> CharacterDatabase.Info = "127.0.0.1;3306;mangos;mangos;characters"
> LogsDatabase.Info      = "127.0.0.1;3306;mangos;mangos;logs"
> ```
> Replace `mangos;mangos` with your actual credentials if you changed them.

---

#### 7. **Add your realm to the database**
```bash
mariadb -u"${DB_USER}" -p"${DB_PASS}" -e "USE realmd; INSERT INTO realmlist (id, name, address, port, icon, realmflags, timezone, allowedSecurityLevel, population, gamebuild_min, gamebuild_max, flag) VALUES (1, 'VMaNGOS', '127.0.0.1', 8085, 1, 0, 1, 0, 0, 5875, 5875, 0);"
```
*Change `'VMaNGOS'` and `'127.0.0.1'` to your desired realm name and IP if needed.*

---

## 5. Starting your server

#### Starting server on Windows
Now that everything is configured, make sure MySQL is running, and simply run _realmd.exe_ and _mangosd.exe_ to start your very own vanilla wow server. Once it is done loading you will hear a beep and you may connect to your server.

#### Starting server on Linux
Open two terminals and run:

**Terminal 1 – Realm server**
```bash
cd ~/vmangos/bin && ./realmd
```
**Terminal 2 – World server**
```bash
cd ~/vmangos/bin && ./mangosd
```
---

## **Optional: Keep servers running with `screen`**
```bash
screen -dmS realmd bash -c "cd ~/vmangos/bin && ./realmd"
screen -dmS mangosd bash -c "cd ~/vmangos/bin && ./mangosd"
```
Reattach with `screen -r realmd` or `screen -r mangosd`.

## **Optional: Keep servers running with `tmux`**
```
tmux new -d -s realmd ~/vmangos/bin/realmd -c ~/vmangos/etc/realmd.conf
tmux new -d -s mangosd ~/vmangos/bin/mangosd -c ~/vmangos/etc/mangosd.conf
```
Reattach with `tmux a -t realmd` or `tmux a -t mangosd`.

### 6. Connecting with client:
To create an admin account, in mangosd type:
```bash
account create admin admin
account set gmlevel admin 6
```
In your wow client folder, open up realmlist.wtf and set it to the following:
```bash
set realmlist 127.0.0.1
```
Once all that is done you can finally login.
