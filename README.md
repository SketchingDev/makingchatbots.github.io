# Dev Meets Contact-Center

## Development

```shell
bundle config set --local path 'vendor/bundle'
```

```shell
bundle install
bundle exec jekyll serve --livereload
```

### Updating Bootstrap

The site uses Bootstrap v5. 

I integrated its SCSS stylesheets by:

1. Check what version is currently being used in `_sass/bootstrap/scss/bootstrap.scss`. There's no point updating to the same version
2. [Download and extract the latest v5.x source files](https://getbootstrap.com/docs/5.0/getting-started/download/#source-files)
3. Delete the contents of the `_sass/bootstrap/` directory 
4. Copy the contents of the `scss/` from the extracted download to `_sass/bootstrap/`

The JS is a compiled version which is updated by:

1. Check what version is currently being used in `assets/js/bootstrap.min.js` to make sure it matches the Bootstrap version above.
2. [Download and extract the compiled version of the release](https://getbootstrap.com/docs/5.0/getting-started/download/#compiled-css-and-js)
3. Copy the following files from the `js` directory in the extracted download to `assets/js/` - overwrite the existing files
   * `bootstrap.min.js`
   * `bootstrap.min.js.map`

## Sketchnotes

### Creating thumbnails

1. Load [Croppola](https://croppola.com/)
2. Expand cropping area to whole image
2. 'Scale to' 500px x 500px

## Troubleshooting

### Problem installing racc

```
Fetching racc 1.5.2
Installing racc 1.5.2 with native extensions
Errno::EACCES: Permission denied @ rb_sysopen - /Users/lucas/sketchingdev.github.io/vendor/bundle/ruby/2.6.0/gems/racc-1.5.2/COPYING
An error occurred while installing racc (1.5.2), and Bundler cannot continue.
Make sure that `gem install racc -v '1.5.2' --source 'https://rubygems.org/'` succeeds before bundling.
```

I resolved this by:
1. [Installing rbenv](https://stackoverflow.com/a/53388305)
2. I didn't have permissions to write to the `./vendor/bundle/ruby/2.6.0/gems/racc-1.5.2` bundle, so I ran:
   ```shell
   cd vendor/bundle/
   sudo chmod -R g+rw *
   ```
3. I could then run the `bundle install` command above

## Useful links

 * [Broken Link Checker](https://ahrefs.com/broken-link-checker)
 * [Backlink Checker](https://ahrefs.com/backlink-checker)
