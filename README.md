# ET-Jekyll
A minimal Jekyll theme inspired by <a href="https://github.com/edwardtufte/tufte-css">Tufte CSS</a>

### Live Demo
A live demo with detailed instructions can be found <a href="https://et-jekyll.netlify.com/et-jekyll-theme/">here</a>.

### Local development

Use Ruby 4.0 and Bundler. Install dependencies inside the repository:

```sh
bundle config set --local path vendor/bundle
bundle install
bundle exec jekyll serve
```

Open http://localhost:4000. The `runblog` shell alias runs the same serve command.

The local build uses Jekyll 4 directly. The `github-pages` dependency bundle
includes older plugins that do not support Ruby 4. GitHub Pages' built-in build
environment uses its own dependency versions; this setup does not change the
hosting configuration.

### Verify it works

```sh
bundle exec jekyll build
curl --fail --output /dev/null --write-out '%{http_code}\n' http://localhost:4000/
```

The build should finish without errors. With the preview server running, the
HTTP check should print `200`. Check that an article displays its code blocks
and that `/style.css` loads.
