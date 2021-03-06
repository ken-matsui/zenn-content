---
title: "GitHub Actions ãŽ if ã¨ env"
emoji: "đ"
type: "tech"
topics: ["github", "githubactions"]
published: true
---

GitHub Actions ãĢéĸããĻããŠããĢ `env` ãæ¸ãããã§ããīŧã¨ãã`if` ãŖãĻãŠãããŖãĻäŊŋããã§ããīŧã¨ãč˛ãã¨čŗĒåãåãããã¨ããããžãã

ããããŖãã`if` ã¨ `env` ãĢéĸããĻããŠãŧãĢãšããĻãããĄããŽč¨äēã§į´šäģãããĻããã ããžãã

# `env`

https://docs.github.com/en/actions/learn-github-actions/environment-variables

ãžãã`env` ãĢéĸããĻã§ãã

## `env` ãäŊŋããå ´æ

`env` ãŽå¯žčąĄį¯å˛ã¯ã`step` åäŊã§ãã

`env` ãæåŽãããĻããã¨ãį°åĸå¤æ°ã¨ããĻã`step` æ¯ãĢåąéãããžãã

```yaml
steps:
  - name: "Say Hello, world!"
    run: echo "$HELLO, world!"
    env:
      HELLO: Hello
```

ã§ã¯äžãã°ãå¨ãĻãŽ `step` ãĢãĻåãį°åĸå¤æ°ãäŊŋį¨ãããæã¯ãäģĨä¸ãŽãããĢ `job` åäŊã§ãåä¸ãŽį°åĸå¤æ°ãåŽįžŠã§ããžãã

```yaml
echo_job:
  runs-on: ubuntu-latest
  env:
    HELLO: Hello
  steps:
    - name: "Say Hello, world!"
      run: echo "$HELLO, world!"
    - name: "Say Hello, foo!"
      run: echo "$HELLO, foo!"
    - name: "Say Hello, bar!"
      run: echo "$HELLO, bar!"
```

ãããĢã`job` ãčˇ¨ãį°åĸå¤æ°ãåŽįžŠã§ããžãã

```yaml
env:
  HELLO: Hello

jobs:
  echo_job:
    runs-on: ubuntu-latest
    steps:
      - name: "Say Hello, world!"
        run: echo "$HELLO, world!"

  hello_job:
    runs-on: ubuntu-latest
    steps:
      - name: "Say Hello, world!"
        run: echo "$HELLO, world!"
```

ããĄã¤ãĢå¨äŊã§ãŽį°åĸå¤æ°åŽįžŠã¯éå¸¸ãĢäžŋåŠã§ããã1 ã¤ãŽ `step` ã§ããäŊŋį¨ããĒãį°åĸå¤æ°ãããŽãããĢåŽįžŠããĻããžããŽã¯č¯ããĒããŽã§éŋããæšãæãžããã§ãã

ãããžã§ãå¯žčąĄį¯å˛ãåēããããåžãĒããĒãŖãæãĢã`job` åäŊãããĄã¤ãĢåäŊãĢåēãããããĢããžãããã

## `job` åã§į°åĸå¤æ°ããģãããã

https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#setting-an-environment-variable

`job` ãŽåŽčĄįĩæãĢåŋããĻãå¤æ°ãåēåããããããåžããį°åĸå¤æ°ã¨ããĻäŊŋį¨ãããæããããžãã

ããŽæã¯ã`GITHUB_ENV` ãĢå¯žããĻã`key=value` ãŽåŊĸåŧã§čŋŊč¨ããã¨ãããŽåžã§ãį°åĸå¤æ°ã¨ããĻãĸã¯ãģãšã§ããžãã

```yaml
steps:
  - name: "Output docker image url"
    run: echo 'DOCKER_IMAGE_URL=gcr.io/cloudrun/hello' >> $GITHUB_ENV

  - name: "Show the docker image url"
    run: echo "$DOCKER_IMAGE_URL"
```

## `env` ã contexts ã¨ããĻäŊŋį¨ãã

https://docs.github.com/en/actions/learn-github-actions/contexts

`${{ github.sha }}` ã§ããŽãŗããããããˇãĨãåããããããŽã§ãããããŽ `github.sha` ãŽé¨åã contexts ãĢããããžãã

äžãã°äģĨä¸ãŽãąãŧãšãčããĻãŋãžãã

```yaml
steps:
  - name: "Output docker image url"
    run: echo 'DOCKER_IMAGE_URL=gcr.io/cloudrun/hello' >> $GITHUB_ENV

  - uses: 'google-github-actions/deploy-cloudrun@v0'
    with:
      service: 'hello-cloud-run'
      image: ''  # į°åĸå¤æ°ãäŊŋããĒãīŧ
```

ããŽæã`with` ã¨ããĻãį°åĸå¤æ°ãæ¸Ąãããããã§ãã

äžãã°äģĨä¸ãŽãããĢããĻããããŽãžãžãŽååŽšãåĨãŖãĻããžããããã¨ãŠãŧãĢãĒãŖãĻããžããžãã

```yaml
      image: '$DOCKER_IMAGE_URL'
```

ããã§ãcontexts ãäŊŋį¨ããã¨ãį°ĄåãĢä¸­čēĢãåąéã§ããžãã

```yaml
      image: '${{ env.DOCKER_IMAGE_URL }}'
```

contexts ã¯ãŠããĒã¨ããã§ãåąéããĻããããŽã§ã`run` ãŽä¸­ãĢæ¸ããĻãå¤§ä¸å¤Ģã§ãã

```yaml
steps:
  - name: "Show the docker image url"
    run: echo '${{ env.DOCKER_IMAGE_URL }}'
```

éãã¯ãbash ãåŽčĄæãĢä¸­čēĢãåąéããããGitHub Actions ãåŽčĄåãĢåąéããããŽéãã§ãã

# `if`

æŦĄã¯ `if` ãĢéĸããĻã§ãã

## `if` ãäŊŋããå ´æ

https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions

`if` ã¯ `env` ã¨éãŖãĻãããĄã¤ãĢåäŊã§ã¯æåŽã§ããžããã

æåŽã§ãããŽã¯ãäģĨä¸ãŽãããĢ `job` åäŊã¨ `step` åäŊãŽãŋã§ãã

```yaml: job åäŊ
jobs:
  assign:
    if: github.actor != 'dependabot[bot]'
```

```yaml: step åäŊ
jobs:
  assign:
    runs-on: ubuntu-latest
    steps:
      - name: Notify actor
        if: github.actor != 'dependabot[bot]'
        run: bash ./notify
```

## `if` ã¯ãĢããŗãįįĨã§ãã

https://docs.github.com/en/actions/learn-github-actions/expressions#about-expressions

`if` ãĢããã `${{ }}` ã¯ãexpression syntax ã¨č¨ããŋããã§ãããããã¯įįĨã§ããžãã

äžãã°ã`dependabot` ãã¯ãŧã¯ãã­ãŧãåŽčĄãããĻãŧãļãŧã§ã¯ãĒãæãĢã`job` ãåŽčĄãããã¨ããĢã¯ãäģĨä¸ãŽãããĢã§ããžãã

```yaml
jobs:
  assign:
    if: github.actor != 'dependabot[bot]'
```

įįĨã§ããã ããĒãŽã§ãäģĨä¸ãŽãããĢæ¸ããĻãåéĄãããžããã

```yaml
jobs:
  assign:
    if: ${{ github.actor != 'dependabot[bot]' }}
```

## `if` ã¨ `env` ãįĩãŋåããã

https://docs.github.com/en/actions/learn-github-actions/expressions#startswith

`if` ã¨ `env` ãįĩãŋåãããĻãŋãžãããã

äģĨä¸ãŽãããĢäŊŋį¨ã§ããžãã

```yaml
steps:
  - name: "Output docker image url"
    run: echo 'DOCKER_IMAGE_URL=gcr.io/cloudrun/hello' >> $GITHUB_ENV

  - uses: 'google-github-actions/deploy-cloudrun@v0'
    if: startsWith(env.DOCKER_IMAGE_URL, 'gcr.io')
    with:
      service: 'hello-cloud-run'
      image: '${{ env.DOCKER_IMAGE_URL }}'
```

## `if` & `env` & `startsWith` & `!`

č˛ãã¨įĩãŋåãããĻãŋãžãããã

```yaml
- uses: 'google-github-actions/deploy-cloudrun@v0'
  if: ${{ !startsWith(env.DOCKER_IMAGE_URL, 'gcr.io') }}
  with:
    service: 'hello-cloud-run'
    image: '${{ env.DOCKER_IMAGE_URL }}'
```

`!` (åĻåŽæŧįŽå­) ã¨åãããĻäŊŋį¨ããæã¯ãä¸č¨ãŽãããĢãexpression syntax ãåŋé ãĢãĒããžãã

ããããŖãčŠąã¯ãäģĨä¸ãĢæ¸ãããĻããžãã

https://github.community/t/if-not-startswith-mutually-exclusive-steps/141841/2

# æåžãĢ

č˛ãã¨į´°ããã¨ããã§ãčŋˇãã¨ãããããã¨æããžãã

įšãĢ expression ãĢéĸããĻã¯ãsyntax ãå°ãįšæŽãĒãŽã§ãæŖãããžã§å°ãæéããããå°čąĄã§ãã

ãžããäģĨä¸ãŽč¨äēã§į´šäģãããĻããã ãã `actionlint` ãäŊŋį¨ããã°ãäēåãĢããããŖãé¨åããã§ãã¯ãã§ããžãã

https://zenn.dev/matken/articles/5-github-actions-for-running-start

ããŽå ´åã¯ãGitHub Actions ã¨ããĻã§ã¯ãĒããã­ãŧãĢãĢã§ `actionlint` ãåŽčĄããã¨ push ããåãĢããšãįēčĻã§ãããŽã§åŽåŋã§ãã

ããĄããŽč¨äēãåčãĢãĒãã°åš¸ãã§ãã
