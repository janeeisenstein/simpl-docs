# simpl-docs provides Simpl framework documentation

It uses the mkdocs package (https://mkdocs.readthedocs.io) to generate static project documentation with Markdown.

### Installation

    $ pip install -r requirements.txt

### Serve the docs

    $ mkdocs serve

Then point your browser at `http://localhost:8000`

### Serve the docs while running a Simpl app locally on localhost:8000

	$ mkdocs serve --dev-addr=0.0.0.0:3000

Then point your browser at `http://localhost:3000`

### Build the docs

To build a static version of the docs, run:

    $ mkdocs build

The static files will be built in the `site` directory.

### TODO

* document errors in simpl store
