---
title: "Migrating/Merging View History between two Plex Servers - Avoiding Negative Unwatched Count"
date: "2019-04-07"
categories: 
  - "backend"
tags: 
  - "database"
  - "plex"
coverImage: "Migrating-plex-watch-history1.jpg"
---

# {{ title }}

<small>Written: {{ date }}</small>

<small>Tags</small>
{% for tag in tags %}
<p style="display:inline">
<a style="padding: .125em 1em; border-radius: 25px; margin-top:5px;" class="md-button md-button--primary" href="#">{{ tag }}</a>
</p>
{% endfor %}

<small>Category</small>
{% for cat in categories %}
<p style="display:inline;">
<a style="padding: .125em 1em; border-radius: 25px; margin-top:5px;" class="md-button md-button--primary" href="#">{{ cat }}</a>
</p>
{% endfor %}

<img src="images/{{ coverImage}}"></img>

Every so often you may put yourself in a situation where you're forced to reinstall Plex. Regrettably, Plex has no built-in functionality for database back-up and migration. This may seem logical, as it is typically unimportant to backup and restore posters and folder matches. However, there is one piece of database information that is worth restoring from previous servers: the **view history**. This information tells the Plex Media Server instance what has been marked as Played/Unplayed. It also contains some extra information such as user ratings.

But the issue arises when merging into a server that isn't a clean install (absolutely no watch history). In this scenario, you may be faced with errors such as **`Error: near line 2319: UNIQUE constraint failed:metadata_item_settings.id`**, and you may see negative unplayed counts within Plex.

![images/673a3fb5910abb54f59c3cca3d5719e53b5be9f72](images/673a3fb5910abb54f59c3cca3d5719e53b5be9f72.jpg)

Migrating watch history while avoiding these issues can be done in six easy steps.

## Part One: Merging Databases

**1) Locate the databases directory** for the Plex install that contains the view history you wish to save. This will be found in **`./Plex Media Server/Plug-in Support/Databases/`**

The default location of this Plex Media Server folder will vary depending on operating system. For more directories see [Where is the Plex Media Server data directory located?](https://support.plex.tv/articles/202915258-where-is-the-plex-media-server-data-directory-located/) Here's the location for a few common operating systems.

Windows: **`%LOCALAPPDATA%`** OSX:**`~/Library/ApplicationSupport/`** Linux: **`$PLEX_HOME/Library/ApplicationSupport/`** Debian: **`/var/lib/plexmediaserver/Library/Application Support/`** FreeBSD: **`/usr/local/plexdata/`** FreeNAS: **`/var/db/plexdata/`** Unraid Docker (Binhex): **`/mnt/cache/appdata/binhex-plex/`**

When you locate this directory, navigate to it via a terminal. This can be done with _cd_ on Linux and _dir_ on Windows.

**2) Ensure that sqlite3 is installed** on the host operating system by typing **`sqlite3 -version`** into your terminal. If an error shows indicating that it is not installed, install SQLite3 through your package manager or through [this link](https://www.sqlite.org/download.html).

**3)** **Put the view history into a file called viewhistory.sql.** This can be done via typing the following command while in the Databases directory:

```bash
echo ".dump metadata_item_settings" | sqlite3 com.plexapp.plugins.library.db | grep -v TABLE | grep -v INDEX > viewhistory.sql
```

**4) Move viewhistory.sql to the databases directory** for the Plex install that you're migrating data into.

**5) Merge view history data from the old server into the new server.** This can by done by running the following:

```bash
cat viewhistory.sql | sqlite3 com.plexapp.plugins.library.db
```

Do not be alarmed if you see a _`UNIQUE constraint failed`_error. This will be fixed in the next step.

## Part Two: Removing the Duplicates

**6) Remove duplicate database entries** that cause negative a view count. Please note, it is highly recommended to create a backup of **`com.plexapp.plugins.library.db`** before attempting to modify it.

If the server you're merging view states to is not perfectly clean (a server with no watch history), or if you received **`UNIQUE constraint failed`** in the previous steps then you must run the following command in the current directory:

```sql
sqlite3 com.plexapp.plugins.library.db
"DELETE FROM metadata_item_settings
WHERE id in (SELECT MIN(id) 
FROM metadata_item_settings 
GROUP BY guid HAVING COUNT(guid) > 1);"
```
