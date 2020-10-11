# How to make Github Actions skip commits which contain `[skip ci]` in message;


Super simple:
```
jobs:
  jobName:
    if: ${{ github.ref != 'refs/heads/master' && !contains(github.event.head_commit.message, '[skip ci]') }} # public; ignore commits with `[skip ci]`
```
