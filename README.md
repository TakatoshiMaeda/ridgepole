# Ridgepole

Ridgepole is a tool to manage DB schema.

It defines DB schema using [Rails DSL](http://guides.rubyonrails.org/migrations.html#types-of-schema-dumps), and updates DB schema according to DSL.
(like Chef/Puppet)

[![Gem Version](https://badge.fury.io/rb/ridgepole.png)](http://badge.fury.io/rb/ridgepole)
[![Build Status](https://travis-ci.org/winebarrel/ridgepole.svg?branch=master)](https://travis-ci.org/winebarrel/ridgepole)
[![Coverage Status](https://coveralls.io/repos/winebarrel/ridgepole/badge.png?branch=master)](https://coveralls.io/r/winebarrel/ridgepole?branch=master)

**Notice**

* `>= 0.4.8`
  * `activerecord-mysql-unsigned` is now optional. Please pass `--enable-mysql-unsigned` after you install [activerecord-mysql-unsigned](https://github.com/waka/activerecord-mysql-unsigned) if you want to use.
  * Please pass `--enable-foreigner` after you install [foreigner](https://github.com/matthuhiggins/foreigner) if you want to use the foreign key.

## Installation

Add this line to your application's Gemfile:

    gem 'ridgepole'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install ridgepole

## Help
```
Usage: ridgepole [options]
    -c, --config CONF_OR_FILE
    -E, --env ENVIRONMENT
    -a, --apply
    -m, --merge
    -f, --file FILE
        --dry-run
        --table-options OPTIONS
        --bulk-change
        --default-int-limit LIMIT
        --pre-query QUERY
        --post-query QUERY
    -e, --export
        --split
        --split-with-dir
    -d, --diff DSL1 DSL2
        --reverse
        --with-apply
    -o, --output FILE
    -t, --tables TABLES
        --ignore-tables TABLES
        --enable-mysql-unsigned
        --enable-foreigner
        --log-file LOG_FILE
        --verbose
        --debug
    -v, --version
```

## Usage

```sh
$ git init
Initialized empty Git repository in ...

$ cat config.yml
adapter: mysql2
encoding: utf8
database: blog
username: root

$ ridgepole -c config.yml --export -o Schemafile # or `ridgepole -c '{adapter: mysql2, database: blog}' ...`
Export Schema to `Schemafile`

$ cat Schemafile
create_table "articles", force: true do |t|
  t.string   "title"
  t.text     "text"
  t.datetime "created_at"
  t.datetime "updated_at"
end

$ git add .
$ git commit -m 'first commit'  -a
[master (root-commit) a6c2d31] first commit
 2 files changed, 10 insertions(+)
 create mode 100644 Schemafile
 create mode 100644 config.yml

$ vi Schemafile
$ git diff
diff --git a/Schemafile b/Schemafile
index f5848b9..c266fed 100644
--- a/Schemafile
+++ b/Schemafile
@@ -1,6 +1,7 @@
 create_table "articles", force: true do |t|
   t.string   "title"
   t.text     "text"
+  t.text     "author"
   t.datetime "created_at"
   t.datetime "updated_at"
 end

$ ridgepole -c config.yml --apply --dry-run
Apply `Schemafile` (dry-run)
add_column("articles", "author", :text, {:after=>"text"})

# ALTER TABLE `articles` ADD `author` text AFTER `text`

$ ridgepole -c config.yml --apply
Apply `Schemafile`
-- add_column("articles", "author", :text, {:after=>"text"})
   -> 0.0202s
```

## Rename
```ruby
create_table "articles", force: true do |t|
  t.string   "title"
  t.text     "desc", renamed_from: "text"
  t.text     "author"
  t.datetime "created_at"
  t.datetime "updated_at"
end

create_table "user_comments", force: true, renamed_from: "comments" do |t|
  t.string   "commenter"
  t.text     "body"
  t.integer  "article_id"
  t.datetime "created_at"
  t.datetime "updated_at"
end
```

## Foreign Key
You can use the foreign key by passing `--enable-foreigner` ([foreigner](https://github.com/matthuhiggins/foreigner) is required)

```ruby
create_table "parent", force: true do |t|
end

create_table "child", id: false, force: true do |t|
  t.integer "id"
  t.integer "parent_id"
end

add_index "child", ["parent_id"], name: "par_ind", using: :btree

add_foreign_key "child", "parent", name: "child_ibfk_1", dependent: :delete
```


## Execute
```ruby
create_table "authors", force: true do |t|
  t.string "name", null: false
end

create_table "books", force: true do |t|
  t.string  "title",                     null: false
  t.integer "author_id", unsigned: true, null: false
end

add_index "books", ["author_id"], name: "idx_author_id", using: :btree

execute("ALTER TABLE books ADD CONSTRAINT fk_author FOREIGN KEY (author_id) REFERENCES authors (id)") do |c|
  # Execute SQL only if there is no foreign key
  c.raw_connection.query("SELECT 1 FROM information_schema.key_column_usage WHERE TABLE_SCHEMA = 'bookshelf' AND CONSTRAINT_NAME = 'fk_author' LIMIT 1").each.length.zero?
end
```

## Diff
```sh
$ ridgepole --diff file1.schema file2.schema
add_column("articles", "author", :text, {:after=>"title"})
rename_column("articles", "text", "desc")

# You can apply to the database the difference:
# $ ridgepole -c config.yml --diff file1.schema file2.schema --with-apply
```

You can also compare databases and files.

```sh
$ ridgepole --diff config.yml file1.schema
remove_column("articles", "author")
```

### Reverse diff
```sh
$ cat file1.schema
create_table "articles", force: true do |t|
  t.string   "title"
  t.text     "text"
  t.datetime "created_at"
  t.datetime "updated_at"
end

$ cat file2.schema
create_table "articles", force: true do |t|
  t.string   "title"
  t.text     "desc", renamed_from: "text"
  t.text     "author"
  t.datetime "created_at"
  t.datetime "updated_at"
end

$ ridgepole --diff file1.schema file2.schema
add_column("articles", "author", :text, {:after=>"title"})
rename_column("articles", "text", "desc")

$ ridgepole --diff file1.schema file2.schema --reverse
rename_column("articles", "desc", "text")
remove_column("articles", "author")
```

## Demo

* [asciinema.org/a/9349](https://asciinema.org/a/9349)
* [asciinema.org/a/11788](https://asciinema.org/a/11788)

## Example project

* https://github.com/winebarrel/ridgepole-example
  * https://github.com/winebarrel/ridgepole-example/pull/1
  * https://github.com/winebarrel/ridgepole-example/pull/2
