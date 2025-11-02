# Internet-Draft Template Repository

Use this repository as a template if you want to start working on
[IETF](https://www.ietf.org/) documents. [Click here to create a new repository using the
template](https://github.com/martinthomson/internet-draft-template/generate).
Make sure to check "Include all branches", or you will need to enable GitHub Pages manually.

[Read the
instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/TEMPLATE.md)
for more information.

Once you have created your own repository, start work by
[renaming the `draft-todo-yourname-protocol.md` file](../../edit/main/draft-todo-yourname-protocol.md).

## Command Line Usage

Command line usage requires that you have the necessary software installed.  See
[the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

Make the current version of the draft with:

```sh
$ make
```

Assuming the previous version has been tagged, make the next numbered version of the draft with:

```sh
$ make next
```

Build the `gh-pages` branch with:

```sh
$ make gh-pages
```

Show unpushed commits on the `gh-pages` branch with:

```sh
$ git --no-pager log --oneline origin/gh-pages..gh-pages
```

Push the `gh-pages` branch to GitHub with:

```sh
$ git push origin gh-pages
```
