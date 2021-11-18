# flutter-optimizing-note

## 기본

적어도 `GetX`같은 상태 관리자를 사용하면 아래 서술한 문제들을 자연스럽게 피할 수 있다.

## 자주겪는 문제

### function-call to stateless-widget

`widget`을 생성할 때 다음과 같이 함수를 사용하는 경우가 있다.

```dart
...
       Column childs: [
                TagInfoAreaWidget(queryResult: data.queryResult),
                DividerWidget(),
                _CommentArea(
                  headers: data.headers,
                  queryResult: data.queryResult,
                ),
                _buildDivider(),
                _previewExpadable(data.queryResult, context)
                DividerWidget(),
                ExpandableNotifier(
                  child: Padding(
                    padding: const EdgeInsets.symmetric(vertical: 4.0),
                    child: ScrollOnExpand(
                      scrollOnExpand: true,
                      scrollOnCollapse: false,
                      child: ExpandablePanel(
                        theme: ExpandableThemeData(
                            iconColor:
                                Settings.themeWhat ? Colors.white : Colors.grey,
                            animationDuration:
                                const Duration(milliseconds: 500)),
                        header: Padding(
                          padding: const EdgeInsets.fromLTRB(12, 12, 0, 0),
                          child:
                              Text(Translations.of(context).trans('preview')),
                        ),
                        expanded:
                            PreviewAreaWidget(queryResult: data.queryResult),
                      ),
                    ),
                  ),
                ),
...
  _previewExpadable(queryResult, context) {
    return ExpandableNotifier(
      child: Padding(
        padding: EdgeInsets.symmetric(vertical: 4.0),
        child: ScrollOnExpand(
          scrollOnExpand: true,
          scrollOnCollapse: false,
          child: ExpandablePanel(
            theme: ExpandableThemeData(
                iconColor: Settings.themeWhat ? Colors.white : Colors.grey,
                animationDuration: const Duration(milliseconds: 500)),
            header: Padding(
              padding: EdgeInsets.fromLTRB(12, 12, 0, 0),
              child: Text(Translations.of(context).trans('preview')),
            ),
            expanded: _previewArea(queryResult),
          ),
        ),
      ),
    );
  }
...
```

위 코드를 보면 `Column-child`의 중간에 `_previewExpandable` 함수를 호출하는 것을 볼 수 있다.
`build` 호출에 의해 플러터 위젯 트리가 생성되고 나면 상태가 변하기 전까진 `build`가 다시 호출되지는 않는다.
하지만 상태가 변하여 `build`가 다시 호출될 경우, 플러터는 기존 위젯 트리와 현재의 위젯 트리를 비교하여 다른 부분 `diff`를 찾는다.
함수에 의해 생성된 위젯 서브 트리의 경우 위젯 트리의 추적을 받지 못하여 플러터의 렌더러는 함수에 의해 생성된 위젯을 다시 그리게된다.
이 문제를 해결하기 위해선 `Widget` 클래스를 사용하는 방법이 있다. `StatelessWidget` 이나 `StatefulWidget`을 사용하면 된다.

## 최적화 테크닉

### Should Reload

`Should Reload`는 프로그래머가 필요할 때만 위젯을 업데이트 할 수 있게 강제하는 방법이다.

```dart
  bool _shouldReload = false;
  Widget _cachedList;

  @override
  Widget build(BuildContext context) {
    final width = MediaQuery.of(context).size.width;
    final height =
        MediaQuery.of(context).size.height - MediaQuery.of(context).padding.top;
    final mediaQuery = MediaQuery.of(context);
    
    // final list = buildList();

    if (_cachedList == null || _shouldReload) {
      final list = buildList();
      _shouldReload = false;
      _cachedList = list;
    }

    return Padding(
      // padding: EdgeInsets.all(0),
      padding: EdgeInsets.only(
          top: MediaQuery.of(context).padding.top,
          bottom: (mediaQuery.padding + mediaQuery.viewInsets).bottom),
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        crossAxisAlignment: CrossAxisAlignment.center,
        children: <Widget>[
          Card(
            elevation: 5,
            color:
                Settings.themeWhat ? Color(0xFF353535) : Colors.grey.shade100,
            child: SizedBox(
              width: width - 16,
              height: height -
                  16 -
                  (mediaQuery.padding + mediaQuery.viewInsets).bottom,
              child: Container(
                child: Padding(
                  padding: EdgeInsets.fromLTRB(0, 0, 0, 0),
                  child: CustomScrollView(
                    physics: const BouncingScrollPhysics(),
                    slivers: <Widget>[
                      SliverPersistentHeader(
                        floating: true,
                        delegate: AnimatedOpacitySliver(
                          minExtent: 64 + 12.0,
                          maxExtent: 64.0 + 12,
                          searchBar: Padding(
                            padding: EdgeInsets.symmetric(horizontal: 8.0),
                            child: Stack(
                              children: <Widget>[
                                _align(),
                                _title(),
                              ],
                            ),
                          ),
                        ),
                      ),
                      buildList()
                      _cachedList
                    ],
                  ),
                ),
              ),
            ),
          ),
        ],
      ),
    );
  }
```

업데이트 방법은 다음과 같다.

```dart
 setState(() {
    _shouldReload = true;
 });
```

```dart
    if (_cachedList == null || _shouldReload) {
      final list = buildList();
      _shouldReload = false;
      _cachedList = list;
    }
```

이 부분에서 buildList 함수를 `_shouldReload` 체커의 안쪽에 놓은 이유는 `build`가 짧은 시간에 여러번 호출될 경우,
플러터 렌더러가 `buildList()`로 생성된 위젯을 다시 그리지는 않지만, 인스턴스 생성 오버헤드가 있기 때문이다.

## 주제들

### 웹툰 형태 이미지 뷰어 구현

파일을 한 장씩 넘겨보는 방식이 아니라 스크롤을 내려서 모든 이미지를 끊임없이 보는 방식을 말한다.
플러터에서는 이걸 구현하는게 매우 어려운데, 그 이유는 플러터 이미지는 메모리 누수가 너무 심하기 때문이다.
그래서 플러터의 2020년 개발 목표를 이미지가 사용하는 메모리를 개선하는 내용도 포함되어 있었다.
나는 이 웹툰 뷰어를 안정적으로 구현하기 위해 많은 방법을 사용했으며, 이제 그 방법을 소개한다.



## Reference

https://blog.codemagic.io/how-to-improve-the-performance-of-your-flutter-app./

https://github.com/rrousselGit/functional_widget

https://medium.com/@remirousselet/flutter-reducing-widgets-boilerplate-3e635f10685e

https://stackoverflow.com/questions/44379366/why-does-setstate-take-a-closure

https://stackoverflow.com/questions/52249578/how-to-deal-with-unwanted-widget-build

https://developpaper.com/the-ultimate-solution-to-prevent-widget-rebuild-by-flutter/
