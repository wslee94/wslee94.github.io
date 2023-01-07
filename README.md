<div align="center">

# wslee94의 블로그

잘 까먹는 나를 위한 공간

블로그 링크: <https://wslee94.github.io>

</div>

## 왜 블로그를 개설했지? (Keep in mind)

- 지식 창고
- 알 것 같은 지식을 알고있는 지식으로 변화
- 브랜딩

## 포스트 작성법 (Keep in mind)

- 왜 사용하는 지 설명하기
- 어떻게 사용하는 지 설명하기
- 가능하다면, 구현 결과를 눈으로 확인할 수 있게 끔 참고자료 넣어두기 (캡처, URL 등)
- 작성 중 다른 기술들이 언급되면 간략히 살명하기
- 정리 (느낀점)

## 개발 환경 세팅

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
