<div align="center">

# wslee94의 블로그

잘 까먹는 나를 위한 공간

블로그 링크: <https://wslee94.github.io>

</div>

## 개발 환경

- ruby 3.0.5
- gem 3.2.33

```console
$ brew install chruby ruby-install xz
$ ruby-install ruby 3.0.5
```

```console
$ echo "source $(brew --prefix)/opt/chruby/share/chruby/chruby.sh" >> ~/.zshrc
$ echo "source $(brew --prefix)/opt/chruby/share/chruby/auto.sh" >> ~/.zshrc
$ echo "chruby ruby-3.1.2" >> ~/.zshrc # run 'chruby' to see actual version
```

## 로컬 서버 실행

```console
$ bundle
$ bundle exec jekyll s
```

<http://localhost:4000>
