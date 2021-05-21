# AmberScript Transcription API Documentation

This repository contains the documentation for the AmberScript Transcription API

This website is powered by [slate](https://github.com/slatedocs/slate).

## Publishing new changes to documentation website

- Stage and commit your changes e.g.:
```
$ git commit -m "Update documentation"
```

- Push the changes to GitHub.
```
$ git push
```

- All changes to `master` branch are automatically deployed using Github Actions. No further action needed!
- If you want to deploy another branch manually, run this locally:
```
$ ./deploy.sh
```

- The docs are live and deployed here: https://amberscript.github.io/api-docs-real-time