# How to make Github Actions skip commits which contain `[skip ci]` in message;

Super simple:

```
jobs:
  jobName:
    if: ${{ github.ref != 'refs/heads/master' && !contains(github.event.head_commit.message, '[skip ci]') }} # public; ignore commits with `[skip ci]`
```

Source: https://github.com/kiprasmel/turbo-schedule/blob/f8d6cbc58a41c3c80bb1c55b41f5b6219bb3521d/.github/workflows/main.yaml
