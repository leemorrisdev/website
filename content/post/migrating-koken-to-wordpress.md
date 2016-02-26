+++
date = "2016-02-26T14:56:00Z"
tags = ["Development"]
title = "Migrating Koken to Wordpress using groovy"

+++

For the last few days I've been working on an upgrade to a website that I maintain for a family member.  As part of this upgrade I wanted to move away from Koken, a portfolio app the site uses currently.  Koken is really nice to use, but comes with a few headaches from an administration perspective.  Namely, it doesn't run on a bog standard shared hosting platform, and it's proven pretty tricky to debug in the past.  I've spent many an hour digging through the codebase trying to figure out why images won't upload, or won't show up on the site.  I took the opportunity to move to Wordpress, more of a known quantity for me.

Aside from building a new theme, I had to migrate the various albums and blog posts from the existing site into wordpress.  The number of albums/posts fell into a grey area between just doing it by hand, and fully automating the entire process.  What I ended up with was a partial automation which I'll detail here.

## Getting started

The first thing I did was to take my last weekly backup (you backup your sites, right?) and import the database into my local MySQL.  This enabled me to build queries and try to reason about how the data was stored in Koken.

## Blog posts

Blog posts in Koken are stored in the **text** table.  The blog posts were pretty heavy on imagery (not surprising for a photographer's site) which, rather unhelpfully, weren't regular img tags, but a shortcode that leaves it to Koken to figure out where the image is actually stored:

    [koken_photo label="IMG2843.jpg" id="698" media_type="image"]

There isn't one big flat directory that stores all images, but rather a storage/originals directory containing dozens of directories that look to use two hex digits, which then also contain dozens of directories using their own two hex digits.  I'm sure there's a reason for this, but it doesn't make it easy to find the image you're looking for.

To get around this I wrote a naive but effective script that would read the text table, find all instances of the koken_photo code, and replace it with an img tag.  On top of this, the script has to find the actual image and copy it somewhere accessible.  Luckily Koken preserves the filename, so I simply search the storage directory for a matching image, and copy it into an import directory that can be copied to the wordpress uploads directory.  Then it's just a case of making sure my img tags are prefixed with "/uploads/imported".

The script dumps the new blog text with the image tags to stdout.  If I had hundreds of blog posts I could have automated the insertion of posts into wordpress, but in my case I had maybe 30 posts so it was quicker to just create them by hand.

    import groovy.sql.Sql

    import java.nio.file.Paths
    import java.nio.file.Files

    def sql = Sql.newInstance("jdbc:mysql://localhost:3306/koken_exported", "root","password", "com.mysql.jdbc.Driver")

    def originalImagesPath = "/path/to/storage/originals"

    def directory = new File(originalImagesPath)
    def uploadPrefix = "/wp-content/uploads/import"

    println "Starting migration"

    def rows = sql.rows("select * from koken_text")

    rows.each { row ->

        def createdOn = ((long)row.created_on) * 1000
        def content = row.content

        // use regex to find all image tags.
        def images = content.findAll("\\[koken_photo label=\"(.*?)\\]")

        images.each { image ->

            // Pull the filename out of the koken_photo tag, stripping quotes as we go
            def fileName = image.find("\"(.*?)\"").replace("\"", "")
            def fullPath = null

            // Search the originals folder to find where this image actually lives
            directory.eachFileRecurse({

                if(it.name == fileName) {
                    fullPath = it.absolutePath
                }
            })

            // We may encounter duplicate files in our target directory, so ensure we're using a unique filename.
            def uniqueFile = uniqueFileName(fileName)

            def sourcePath = Paths.get(fullPath)
            def targetPath = Paths.get("./images/" + uniqueFile)

            Files.copy(sourcePath, targetPath)

            // replace the koken photo label with the img tag
            content = content.replace(image, '<img src="' + uploadPrefix + "/" + uniqueFile + '" />')
        }

        println "\n\n================\nPOST\n================"
        println "Created: " + new Date(createdOn)
        println content

    }

    //
    // Generates a unique filename, makes recursive calls until a file is not found.
    //
    def uniqueFileName(fileName) {

        def targetPath = Paths.get("./images/$fileName")

        if(Files.exists(targetPath)) {

            def newFileName

            if(fileName.contains("_import_")) {
                def version = Integer.valueOf(fileName.substring(0,1)) + 1

                newFileName = version + fileName.substring(1)
            } else {
                newFileName = "1_import_$fileName"
            }

            return uniqueFileName(newFileName)
        }

        return fileName

    }

The script took a few seconds to run.  Afterwards I copied the images directory into wordpress uploads, and copy/pasted the blog entries.

## Albums

Migrating albums were considerably easier.  Koken uses several tables to map the one->many relationship between album and photograph.  The aptly named album table holds the details of the albums themselves, the content table holds a reference to an individual image (not just for albums but other things too).  The two tables are joined by the koken_join_albums_content table.

In this case, I want to create separate directories for each album and copy the images into them.  Then, I can create the corresponding albums using whatever wordpress plugin I use, and upload each directory in turn.  Again, if I had hundreds of albums I might have automated the entire process, but in my case I had approx. 150 images across 6 albums so partial automation was fine.

The query to get a list of albums->images is as follows:

    select a.title, c.filename from koken_albums a
    left join koken_join_albums_content j on j.album_id = a.id
    left join koken_content c on c.id = j.content_id

There are many fields available but in my case I only want a mapping of filename to album title.

The script is as follows:

    import groovy.sql.Sql

    import java.nio.file.Paths
    import java.nio.file.Files

    def sql = Sql.newInstance("jdbc:mysql://localhost:3306/koken_exported", "root","password", "com.mysql.jdbc.Driver")

    def originalImagesPath = "/path/to/storage/originals"
    def directory = new File(originalImagesPath)

    def rows = sql.rows("""select a.title, c.filename from koken_albums a
    left join koken_join_albums_content j on j.album_id = a.id
    left join koken_content c on c.id = j.content_id""")

    rows.each { row ->

        def albumName = row.title
        def fileName = row.filename

        def fullPath = null

        directory.eachFileRecurse({
            if(it.name == fileName) {
                fullPath = it.absolutePath
            }
        })

        new File("./albums/$albumName").mkdirs()

        def sourcePath = Paths.get(fullPath)
        def targetPath = Paths.get("./albums/$albumName/$fileName")

        println "Copying $sourcePath to $targetPath"

        Files.copy(sourcePath, targetPath)

    }

Once the script was run, I had a directory full of albums that I could then drag/drop into wordpress.

## Thoughts

This was a quick and dirty way to migrate the data, there are several places it could be improved.

1. Use wordpress API to automate the rest of it - not needed in my case but I'd look at this for a large migration.
2. Handle potential cases of missing files - it could happen, but in the interests of speed I didn't worry about this so much and it didn't bite me in the backside for this task.
3. Handle potential duplicate filenames - we do this in the target directory, but then it's possible the source directory could have duplicates and we may end up using the wrong image.
