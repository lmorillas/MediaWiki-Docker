# MediaWiki Docker Image

Based on MediaWiki release [1.27](https://www.mediawiki.org/wiki/Release_notes/1.27).
Requires Docker version 1.11.2.

Installed extensions (34 in total):

- [Cargo](https://www.mediawiki.org/wiki/Extension:Cargo), v1.0.1
- [CirrusSearch](https://www.mediawiki.org/wiki/Extension:CirrusSearch)
- [Cite](https://www.mediawiki.org/wiki/Extension:Cite)
- [CodeEditor](https://www.mediawiki.org/wiki/Extension:CodeEditor)
- [Collection](https://www.mediawiki.org/wiki/Extension:Collection)
- CustomParser
- [DynamicPageList](https://www.mediawiki.org/wiki/Extension:DynamicPageList_(third-party))
- [Echo](https://www.mediawiki.org/wiki/Extension:Echo)
- [Editcount](https://www.mediawiki.org/wiki/Extension:Editcount)
- [Elastica](https://www.mediawiki.org/wiki/Extension:Elastica)
- [Embed](https://www.mediawiki.org/wiki/Extension:SimpleEmbed)
- [External Data](https://www.mediawiki.org/wiki/Extension:External_Data)
- [Flow](https://www.mediawiki.org/wiki/Extension:Flow)
- [Gadgets](https://www.mediawiki.org/wiki/Extension:Gadgets)
- [InputBox](https://www.mediawiki.org/wiki/Extension:InputBox)
- [Interwiki](https://www.mediawiki.org/wiki/Extension:Interwiki)
- [LookupUser](https://www.mediawiki.org/wiki/Extension:LookupUser)
- [MultimediaViewer](https://www.mediawiki.org/wiki/Extension:MultimediaViewer)
- [MyVariables](https://www.mediawiki.org/wiki/Extension:MyVariables)
- [News](https://www.mediawiki.org/wiki/Extension:News)
- [ParserFunctions](https://www.mediawiki.org/wiki/Extension:ParserFunctions)
- [ParserHooks](https://www.mediawiki.org/wiki/Extension:ParserHooks)
- [PdfHandler](https://www.mediawiki.org/wiki/Extension:PdfHandler)
- [RandomSelection](https://www.mediawiki.org/wiki/Extension:RandomSelection)
- [Renameuser](https://www.mediawiki.org/wiki/Extension:Renameuser)
- [ReplaceText](https://www.mediawiki.org/wiki/Extension:ReplaceText)
- [Scribunto](https://www.mediawiki.org/wiki/Extension:Scribunto)
- [SemanticForms](https://www.mediawiki.org/wiki/Extension:SemanticForms), v3.6
- [SubPageList](https://www.mediawiki.org/wiki/Extension:SubPageList)
- [SyntaxHighlight_GeSHi](https://www.mediawiki.org/wiki/Extension:SyntaxHighlight), depends on Python
- [UploadWizard](https://www.mediawiki.org/wiki/Extension:UploadWizard)
- [VisualEditor](https://www.mediawiki.org/wiki/Extension:VisualEditor)
- [Widgets](https://www.mediawiki.org/wiki/Extension:Widgets)
- [WikiEditor](https://www.mediawiki.org/wiki/Extension:WikiEditor)

Other features:

- [StringFunctions](https://www.mediawiki.org/wiki/Extension:StringFunctions) are enabled

## Usage

The image is built and uploaded to Docker Hub, so there is no need to clone the repository.
A simple `docker-compose.yml` is enough to get started:

```yaml
version: '2'
services:
  elasticsearch:
    image: initiumlab/mediawiki-elasticsearch:1.7.5-1
  mysql:
    image: mysql:5.7.12
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: mediawiki
    volumes:
      - ./data/dump:/docker-entrypoint-initdb.d
  parsoid:
    image: initiumlab/mediawiki-parsoid:1.26.2-1
  mediawiki:
    image: initiumlab/mediawiki:1.26.2-8
    ports:
      - 80:80
    links:
      - mysql
      - parsoid
    environment:
      WG_SITENAME: MediaWiki
      WG_DBNAME: mediawiki
      WG_SERVER: http://192.168.99.100
      WG_SECRET_KEY: ee0efe6c8b4bc1ee5ccd906ad783aeb20115f061a3f9d85e6850612104920701
      WG_UPGRADE_KEY: 08882e35d74f30cf
    volumes:
      - ./data/images:/var/www/html/w/images
```

### Start a new wiki

Run `docker-compose exec mediawiki bash` to start a bash:

```
$ cd /var/www/html/w
$ mv LocalSettings.php LocalSettings.php.shadow # disable settings so we can ask the installer to setup database
$ php maintenance/install.php --dbname mediawiki --dbserver mysql --dbuser root --dbpass password --server $WG_SERVER --pass password $WG_SITENAME Administrator
$ mv LocalSettings.php.shadow LocalSettings.php # enable extensions
$ php maintenance/update.php --quick
$ php /var/www/html/w/extensions/CirrusSearch/maintenance/updateSearchIndexConfig.php
```

The installation script gives us a root user with name `Administrator` and password `password`.
After setup, run `docker-compose up` to start the server.

### Restore a previous wiki

If you have a previous MediaWiki setup, follow the steps:

- put the database dump file at `data/dump` folder, make sure there is a `CREATE DATABASE` command in your dump file
- put the `images` folder from previous setup to `data/images`
- make sure the `docker-compose.yml` file has the correct database settings
- run `upgrade.php`
- run `docker-compose up`
- rebuild ElasticSearch index

## Maintenance

### Upgrade

If you want to use a newer version of images, pull the corresponding image and change image tag in `docker-compose.yml`.
After that, restart the container:

```bash
$ docker-compose up -d --no-deps mediawiki
```

If database migration is needed, run the `upgrade.php` script:

```
$ docker-compose exec mediawiki php /var/www/html/w/maintenance/update.php
```

### Database backup and restore

To dump the database, run:

```
$ docker-compose exec mysql sh -c 'exec mysqldump initium_wiki -B -uroot -p"password"' > dump.sql
```

To restore database from a MySQL dump file:

- run `docker-compose rm mysql` to delete existing database
- put the dump file at `data/dump` folder
- run `docker-compose up`, the database will be created

### Rebuild ElasticSearch index

If your previous installation does not use ElasticSearch, you need to bootstrap the search index:

```bash
$ php /var/www/html/w/extensions/CirrusSearch/maintenance/forceSearchIndex.php --skipLinks --indexOnSkip
$ php /var/www/html/w/extensions/CirrusSearch/maintenance/forceSearchIndex.php --skipParse
```

## Development

There are two methods to install new extensions:

- Git submodule: put the extension files in `mediawiki/extensions` folder and configure the extension in `settings.d/42-extensions.php` file.
- PHP Composer: add an entry to `docker/mediawiki/composer.local.json`.

If you want to build the image by yourself:

- clone this repo
- `git submodule update --init`.
  - For VisualEditor, you have to run `cd mediawiki/extensions/VisualEditor && git submodule update --init`.
  - For Widgets, you have to run `cd mediawiki/extensions/Widgets && git submodule update --init`.
- `docker-compose build`

## Troubleshooting

### OCG 

To test if the bundler can reach the MediaWiki service and generate bundle file:
```bash
$ /app/mw-ocg-bundler/bin/mw-ocg-bundler --parsoid-api http://parsoid:8000 --php-api http://mediawiki/w --api-version parsoid3 -o bundle.zip -v -h mediawiki "Main Page"
```

To test if the latex renderer is able to render the generated bundle file:

```bash
/app/mw-ocg-latexer/bin/mw-ocg-latexer -l -D -o out.tex --lang ZH bundle.zip
TEXINPUTS=tex/: xelatex out.tex
```

If the MediaWiki server address is not accessible for OCG server, add a line to `/app/mw-ocg-bundler/lib/metabook.js` line 245:

```javascript
metabook.wikis[0]['baseurl'] = "http://mediawiki/w";
```

## License

MIT License
