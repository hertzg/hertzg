# Setup Yarn 2 (berry) per project and Zero-Installs

* Ensure you have latest yarn

```
npm i -g yarn
```

* Make yarn switch to v2 version a.k.a. `berry`

```
yarn set version berry
```

* Update the .gitignore respectively

```
# Ignore Yarn 2 files including Zero-Installs
.yarn/*
!.yarn/cache
!.yarn/releases
!.yarn/plugins
!.yarn/sdks
!.yarn/versions
```

* (optional) Mark generated and vendored `.gitattributes` for GitHub language detector (linguist)

```
# Mark .zip files as binary to prevent git from trying to merge it
*.zip                   binary

# Mark .pnp.* as binary to prevent git from trying to merge it & generated to hide from github linguist
/.pnp.*                 binary linguist-generated

# Hide .yarn from GitHub's language detection
/.yarn/**               linguist-vendored
```
