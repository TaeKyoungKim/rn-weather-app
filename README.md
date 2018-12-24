
{title="src/App.js",lang=javascript}
~~~~~~~~
const DEFAULT_QUERY = 'redux';
# leanpub-start-insert
const DEFAULT_HPP = '100';
# leanpub-end-insert

const PATH_BASE = 'https://hn.algolia.com/api/v1';
const PATH_SEARCH = '/search';
const PARAM_SEARCH = 'query=';
const PARAM_PAGE = 'page=';
# leanpub-start-insert
const PARAM_HPP = 'hitsPerPage=';
# leanpub-end-insert
~~~~~~~~

이 상수를 조합해 URL를 만듭니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
fetchSearchTopStories(searchTerm, page = 0) {
# leanpub-start-insert
  fetch(`${PATH_BASE}${PATH_SEARCH}?${PARAM_SEARCH}${searchTerm}&${PARAM_PAGE}${page}&${PARAM_HPP}${DEFAULT_HPP}`)
# leanpub-end-insert
    .then(response => response.json())
    .then(result => this.setSearchTopStories(result))
    .catch(error => error);
}
~~~~~~~~

이전보다 더 많은 리스트를 가져오게 되었습니다. 이번 절에서는 해커 뉴스 API를 통해 받은 실제 데이터를 사용해봤습니다. 새 프로그래밍 언어나 라이브러리를 학습할 때, 외부 API를 사용하면 더 재미있게 배울 수 있을 겁니다. 저도 이렇게 배웠으니까요.

### 실습하기

* [해커 뉴스 API](https://hn.algolia.com/api)의 매개변수를 바꿔 API를 요청해봅니다.

## 3.7 클라이언트 캐시

"Search" 버튼을 클릭하면 해커 뉴스 API에 요청을 보냅니다. 예를 들어 "redux"를 검색하고, 그다음 "react"를, 그리고 다시 "redux"를 입력해 검색 요청을 보냈다고 가정해봅시다. 총 세 번 검색 요청을 보냈습니다. 이 중 "redux"를 두 번 요청했는데, 두 번 모두 데이터를 가져오기 위해 비동기 왕복 여행(asynchronous roundtrip)을 거치게 됩니다. 이 경우 클라이언트 캐시(client-sided cache)를 도입하는 것이 좋습니다. 캐시(Cache)란 미리 만들어진 데이터를 임시로 저장하는 공간입니다. 우리가 구현할 내용은 다음과 같습니다. 새 API 요청을 받으면, 이전에 동일한 요청이 있는지 먼저 확인합니다. 이미 저장된 캐시가 있으면 이를 사용하고, 캐시가 없다면 새 데이터를 가져오기 위해 API를 요청합니다. 이제 시작해봅시다.

각 결과마다 클라이언트 캐시를 보유하려면, 컴포넌트 내부 상태에 `result` 하나 보다, 여러 `results`를 저장하는 것이 좋습니다. 이를 위해 `results` 객체는 키(key)가 검색어이고, 값(value)이 `hits`가 되게 만들겠습니다. API 결괏값은 검색어(key)에 매핑되어 저장되게 해봅시다.

현재 로컬 상태는 아래 코드와 비슷할 것입니다.

{title="Code Playground",lang="javascript"}
~~~~~~~~
result: {
  hits: [ ... ],
  page: 2,
}
~~~~~~~~

예를 들어, "redux" 그리고 "react"로 두 번 검색 요청을 보낸다면 `results` 객체는 아래와 같을 것입니다.

{title="Code Playground",lang="javascript"}
~~~~~~~~
results: {
  redux: {
    hits: [ ... ],
    page: 2,
  },
  
  react: {
    hits: [ ... ],
    page: 1,
  },
  ...
}
~~~~~~~~

이제 `setState()` 메서드로 클라이언트 캐시를 구현해봅시다. 

첫째, 컴포넌트 내부 상태의 `result` 객체를 `results`로 변경합니다. 둘째, `searchKey`를 정의합니다. `searchKey`는 임시 키(key)로서 `result`를 저장하는데 쓰입니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
class App extends Component {

  constructor(props) {
    super(props);

    this.state = {
# leanpub-start-insert
      results: null,
      searchKey: '',
# leanpub-end-insert
      searchTerm: DEFAULT_QUERY,
    };

    ...

  }

  ...

}
~~~~~~~~

API 요청을 보내기 전, `searchKey`를 설정해야 합니다. `searchKey`은 검색어인 `searchTerm`를 반영합니다. `searchTerm`가 아닌 `searchKey`를 사용하는 이유는 무엇일까요? `searchTerm`은 Search 컴포넌트에서 입력 필드에 검색어를 입력할 때마다 그 값이 변경되는 변경 변수(fluctuant variable)입니다. 값이 변경되지 않는 고정 변수가 필요합니다. 따라서 `searchKey`로 `results` 객체 내 해당하는 결과를 찾습니다. 이 변수는 캐시 내 현재 결과를 가리키는 포인터 역할을 합니다. 따라서 최종적으로 `render()` 메서드에 현재 결과가 표시됩니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
componentDidMount() {
  const { searchTerm } = this.state;
# leanpub-start-insert
  this.setState({ searchKey: searchTerm });
# leanpub-end-insert
  this.fetchSearchTopStories(searchTerm);
}

onSearchSubmit(event) {
  const { searchTerm } = this.state;
# leanpub-start-insert
  this.setState({ searchKey: searchTerm });
# leanpub-end-insert
  this.fetchSearchTopStories(searchTerm);
  event.preventDefault();
}
~~~~~~~~

다음으로 `searchKey`를 사용해 컴포넌트 내부 상태에 각 `result`를 저장하도록 수정하겠습니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
class App extends Component {

  ...

  setSearchTopStories(result) {
    const { hits, page } = result;
# leanpub-start-insert
    const { searchKey, results } = this.state;

    const oldHits = results && results[searchKey]
      ? results[searchKey].hits
      : [];
# leanpub-end-insert

    const updatedHits = [
      ...oldHits,
      ...hits
    ];

    this.setState({
# leanpub-start-insert
      results: {
        ...results,
        [searchKey]: { hits: updatedHits, page }
      }
# leanpub-end-insert
    });
  }

  ...

}
~~~~~~~~

`searchKey`는 업데이트된 `hits`와 `page`를 `results`에 저장하기 위한 키(key)로 사용됩니다. 각 단계별로 자세히 설명하겠습니다.

첫째, 컴포넌트 내부 상태에서 `searchKey`를 받습니다. `searchKey`는 `componentDidMount()`와 `onSearchSubmit()` 메서드에서 설정했습니다.

둘째, `oldhits` 변수는 이미 검색 결과가 있는지 확인하기 위해 `searchKey` 에 해당하는 목록을 찾고 그 값을 저장합니다.

셋째, `updateHits` 변수는 총 결괏값을 전달하기 위해 이전 결과인 `oldhits`와 새 결과인 `hits`를 합칩니다.

 다음으로 `setState()`에 있는 `results` 객체를 살펴봅시다.

{title="src/App.js",lang=javascript}
~~~~~~~~
results: {
  ...results,
  [searchKey]: { hits: updatedHits, page }
}
~~~~~~~~

`  [searchKey]: { hits: updatedHits, page }
` 이 부분을 자세히 설명하겠습니다. `results` 객체에서 `searchKey`로 찾은 결과를 저장함을 말합니다. 키(key)는 `searchKey`는 검색어이며, 값(value)은 `hits`와 `page` 프로퍼티가 있는 객체입니다. 이전 절에서 `[searchKey]:...`와 같은 구문을 이미 배웠습니다. ES6의 계산된 프로퍼티 이름(ES6 computed property name)로 객체에서 값을 동적으로 할당할 때 사용됩니다.

`  ...results,` 부분은 객체 전개 연산자를 사용해 `searchKey`에 따른 모든 `results`를 전파합니다. 그렇지 않으면 이전에 저장된 모든 `results`가 손실됩니다. 

이제 모든 결과를 검색어로 저장하겠습니다. 먼저 캐시를 활성화합니다. 그리고 고정값인 `searchKey`에 따라 `results`를 검색합니다. `searchKey`를 사용하지 않으면 Search 컴포넌트를 사용할 때 그 값이 계속 변경됩니다. 따라서 변동값인 `searchTerm`를 사용할 경우 검색이 중단되는 문제가 발생합니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
class App extends Component {

  ...

  render() {
# leanpub-start-insert
    const {
      searchTerm,
      results,
      searchKey
    } = this.state;

    const page = (
      results &&
      results[searchKey] &&
      results[searchKey].page
    ) || 0;

    const list = (
      results &&
      results[searchKey] &&
      results[searchKey].hits
    ) || [];

# leanpub-end-insert
    return (
      <div className="page">
        <div className="interactions">
          ...
        </div>
# leanpub-start-insert
        <Table
          list={list}
          onDismiss={this.onDismiss}
        />
# leanpub-end-insert
        <div className="interactions">
# leanpub-start-insert
          <Button onClick={() => this.fetchSearchTopStories(searchKey, page + 1)}>
# leanpub-end-insert
            More
          </Button>
        </div>
      </div>
    );
  }
}
~~~~~~~~

`searchKey`에 해당하는 결괏값을 찾지 못할 때, `list`는 비어있습니다. 이렇게 만들면 Table 컴포넌트를 조건부 렌더링으로 처리할 수 있습니다. "More" 버튼에 `searchTerm`를 `searchKey`로 수정했습니다. 이렇게 하지 않으면  `searchTerm`이 변경될 때마다 페이지 매김 데이터를 가져오게 됩니다. Search 컴포넌트 내 입력 필드 값이 변경될 때 마다 `searchTerm`에 저장됨을 잊지 마세요.

검색 기능이 잘 작동하는지, 해커 뉴스 API에서 받은 데이터가 잘 저장되는지 확인하세요.

추가로 `onDismiss()` 메서드도 개선하겠습니다. `result` 객체가 아닌, 여러 `results`를 다루도록 수정하겠습니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
  onDismiss(id) {
# leanpub-start-insert
    const { searchKey, results } = this.state;
    const { hits, page } = results[searchKey];
# leanpub-end-insert

    const isNotId = item => item.objectID !== id;
# leanpub-start-insert
    const updatedHits = hits.filter(isNotId);

    this.setState({
      results: {
        ...results,
        [searchKey]: { hits: updatedHits, page }
      }
    });
# leanpub-end-insert
  }
~~~~~~~~

"Dismiss" 버튼이 잘 동작하는지 확인하세요. 

하지만 캐시 기능이 완벽하게 구현되지 않았습니다. 검색 요청 시, API 요청을 막는 장치가 없습니다. 이미 동일한 결과가 있을 때 API 요청을 방지하는 검사를 하지 않습니다. `results`는 캐시에 저장되지만 이를 사용하지 않았습니다. 마지막으로 캐시에서 `results`를 사용할 수 있을 때, API 요청을 막도록 수정해봅시다.

{title="src/App.js",lang=javascript}
~~~~~~~~
class App extends Component {

  constructor(props) {

    ...

# leanpub-start-insert
    this.needsToSearchTopStories = this.needsToSearchTopStories.bind(this);
# leanpub-end-insert
    this.setSearchTopStories = this.setSearchTopStories.bind(this);
    this.fetchSearchTopStories = this.fetchSearchTopStories.bind(this);
    this.onSearchChange = this.onSearchChange.bind(this);
    this.onSearchSubmit = this.onSearchSubmit.bind(this);
    this.onDismiss = this.onDismiss.bind(this);
  }

# leanpub-start-insert
  needsToSearchTopStories(searchTerm) {
    return !this.state.results[searchTerm];
  }
# leanpub-end-insert

  ...

  onSearchSubmit(event) {
    const { searchTerm } = this.state;
    this.setState({ searchKey: searchTerm });
# leanpub-start-insert

    if (this.needsToSearchTopStories(searchTerm)) {
      this.fetchSearchTopStories(searchTerm);
    }

# leanpub-end-insert
    event.preventDefault();
  }

  ...

}
~~~~~~~~

이제 클라이언트는 동일한 검색어를 두 번 검색하더라도, 단 한번 API 요청을 보냅니다. 페이지 매김 데이터도 같은 방법으로 캐시됩니다. `result`에 항상 마지막 페이지가 저장되기 때문입니다. 

## 3.8 오류 처리 

이전 절에서 API 결과를 캐싱하고 페이지 매김 기능을 구현해 더 많은 기사 리스트를 계속 가져오게 만들었습니다. 그러나 제일 중요한 부분을 하지 않았습니다. 안타깝게도 많은 개발자들이 애플리케이션을 개발하면서 오류 처리(Error Handling)를 간과하고 있습니다. 오류 처리를 무시한 채 개발하는 것은 정말 쉽기 때문이지요.

이번 절에서는 API 요청 시, 효과적인 오류 처리 방법에 대해 알아보겠습니다. 이전 절에서 로컬 상태와 조건부 렌더링으로 오류 처리하는 방법을 배웠습니다. 기본적으로 오류는 리액트에서 또 다른 "상태(state)"라고 볼 수 있습니다. 오류가 발생하면 로컬 상태로 저장하고 컴포넌트의 조건부 렌더링으로 오류 메시지를 표시할 수 있습니다. 우리는 App 컴포넌트의 오류 처리를 다루겠습니다. App 컴포넌트는 해커 뉴스 API로 데이터를 받는 곳이기 주요 컴포넌트이기 때문입니다. 그럼 이제 시작해봅시다.

첫째, 로컬 상태에 오류를 추가합니다. `error` 객체의 초기값은 `null`이고, 오류가 발생하면 오류 메시지 값으로 설정됩니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
class App extends Component {
  constructor(props) {
    super(props);

    this.state = {
      results: null,
      searchKey: '',
      searchTerm: DEFAULT_QUERY,
# leanpub-start-insert
      error: null,
# leanpub-end-insert
    };

    ...
  }

...

}
~~~~~~~~

둘째, fetch API의 `catch()` 블록 안에 `setState()`메서드로 오류 객체를 로컬 상태로 저장합니다. API 요청이 실패하면 catch 블록이 실행됩니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
class App extends Component {

  ...

  fetchSearchTopStories(searchTerm, page = 0) {
    fetch(`${PATH_BASE}${PATH_SEARCH}?${PARAM_SEARCH}${searchTerm}&${PARAM_PAGE}${page}&${PARAM_HPP}${DEFAULT_HPP}`)
      .then(response => response.json())
      .then(result => this.setSearchTopStories(result))
# leanpub-start-insert
      .catch(error => this.setState({ error }));
# leanpub-end-insert
  }

  ...

}
~~~~~~~~

셋째, `render()` 내 로컬 상태 내 `error` 객체를 가져오고 조건문 렌더링으로 오류 메시지를 표시합니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
class App extends Component {
  
  ...

  render() {
    const {
      searchTerm,
      results,
      searchKey,
# leanpub-start-insert
      error
# leanpub-end-insert
    } = this.state;

    ...

# leanpub-start-insert
    if (error) {
      return <p>Something went wrong.</p>;
    }
# leanpub-end-insert

    return (
      <div className="page">
        ...
      </div>
    );
  }
}
~~~~~~~~

여기까지입니다. 오류 처리에 문제 없는지를 확인하려면 API URL을 존재하지 않는 URL로 바꿔서 테스트해보세요.

{title="src/App.js",lang=javascript}
~~~~~~~~
const PATH_BASE = 'https://hn.foo.bar.com/api/v1';
~~~~~~~~

애플리케이션 대신, 오류 메시지가 표시됩니다. 원하는 곳에 조건부 렌더링으로 오류 메시지를 표시하면 됩니다. 조건부 렌더링을 전체 앱으로 감싸게 되면 빈 화면만 보일 것입니다. 따라서 Table 컴포넌트 또는 오류 메시지 둘 중 하나를 표시하면 어떨까요? 오류가 발생해도 나머지 컴포넌트는 화면에 보이기 때문에 사용자에게 혼란을 주지 않을 것입니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
class App extends Component {

  ...

  render() {
    const {
      searchTerm,
      results,
      searchKey,
      error
    } = this.state;

    const page = (
      results &&
      results[searchKey] &&
      results[searchKey].page
    ) || 0;

    const list = (
      results &&
      results[searchKey] &&
      results[searchKey].hits
    ) || [];

    return (
      <div className="page">
        <div className="interactions">
          ...
        </div>
# leanpub-start-insert
        { error
          ? <div className="interactions">
            <p>Something went wrong.</p>
          </div>
          : <Table
            list={list}
            onDismiss={this.onDismiss}
          />
        }
# leanpub-end-insert
        ...
      </div>
    );
  }
}
~~~~~~~~

오류 테스트를 위해 기존 URL를 수정했다면 원 상태로 되돌려야 합니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
const PATH_BASE = 'https://hn.algolia.com/api/v1';
~~~~~~~~

애플리케이션이 잘 작동하는지 확인하세요. 

### 읽어보기

* [[리액트 공식 문서] 리액트 컴포넌트 오류 처리](https://reactjs.org/blog/2017/07/26/error-handling-in-react-16.html)

{pagebreak}

## 3.9 Axios 라이브러리 사용

이전 절에서 fetch API로 해커 뉴스 데이터를 받아왔습니다. 하지만 모든 브라우저가 fetch API를 지원하는 것은 아닙니다. 특히 오래된 브라우저는 fetch API를 지원하지 않으며, 헤드리스 브라우저(Headless Browser)에서 애플리케이션을 테스트할 때, fetch API와 관련된 문제가 발생할 수 있습니다. 헤드리스 브라우저는 실제 애플리케이션이 구동되는 브라우저가 아닌 테스트용으로 테스트 자동화에 사용되는 브라우저입니다. 물론 구버전 브라우저(폴리필, polyfill)와 테스트([isomorphic-fetch](https://github.com/matthew-andrews/isomorphic-fetch))에서도 fetch API를 사용할 수 있는 방법이 있지만, 이 책에서는 이 내용을 다루지 않겠습니다.

이와 같이 브라우저 간 문제와 API의 한계를 해결하고자 기존 fetch API를 대체하는 라이브러리를 도입할 수 있습니다. 그 중 [axios](https://github.com/axios/axios)는 많은 리액트 개발자들이 사용하고 있는 비동기 API 라이브러리입니다. 이번 절에서는 axios 라이브러리 설치와 사용 방법, 웹 개발의 고질적인 문제점인 구버전 브라우저과 헤드리스 브라우저 테스트 해결 방법에 대해 알아보겠습니다.

fetch API를 axios로 바꿔봅시다.

첫째, 커맨드 라인에 axios를 설치합니다.

{title="Command Line",lang="text"}
~~~~~~~~
npm install axios
~~~~~~~~

둘째, App 컴포넌트 파일에 axios를 가져옵니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
import React, { Component } from 'react';
# leanpub-start-insert
import axios from 'axios';
# leanpub-end-insert
import './App.css';

...
~~~~~~~~

지금부터 `fetch()` 메서드 대신 `axios()` 메서드를 사용하겠습니다. 사용법은 fetch API와 거의 동일합니다. 인자는 URL이며 프로미스(promise)를 반환합니다. 반환된 응답을 JSON으로 변환하지 않아도 됩니다. axios 이 일을 하며. 자바스크립트에서 `data` 객체로 반환합니다. 따라서 반환된 데이터 구조 그대로 사용하면 됩니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
class App extends Component {

  ...

  fetchSearchTopStories(searchTerm, page = 0) {
# leanpub-start-insert
    axios(`${PATH_BASE}${PATH_SEARCH}?${PARAM_SEARCH}${searchTerm}&${PARAM_PAGE}${page}&${PARAM_HPP}${DEFAULT_HPP}`)
      .then(result => this.setSearchTopStories(result.data))
# leanpub-end-insert
      .catch(error => this.setState({ error }));
  }

  ...

}
~~~~~~~~

`axios()` 메서드는 디폴트로 HTTP GET 요청을 보냅니다. HTTP GET 요청을 좀더 명확하게 표현하려면 `axios.get()` 메서드를 사용합니다. 마찬가지로 HTTP POST 요청은 `axios.post()` 메서드를 사용합니다. 이처럼 axios는 간단한 코드만으로 원격 API 요청을 훌륭하게 수행합니다. 기존 fetch API는 코드가 복잡하며 프로미스(promise)도 처리해야 해서 불편했지만, axios로 개발이 편하고 쉽게 개발할 수 있습니다. 특정 브라우저와 헤드리스 브라우저 환경 문제 역시 해결됩니다. 

App 컴포넌트에서 해커 뉴스 API 요청 시, 고쳐야할 점이 하나 더 있습니다. 예를 들어 `componentDidMount()` 생명주기 메서드에서 API 요청을 하는 동안, 네비게이션을 클릭해 다른 페이지로 이동했다고 가정해봅시다. 그리고 `then()` 또는 `catch()` 에서 `this.setState()` 메서드를 사용했습니다. 컴포넌트 마운트가 해제되더라도 여전히 `componentDidMount()` 메서드에서는 보류 중인 요청이 존재합니다. 이 경우 브라우저 개발자 도구나 커맨드 라인에서 아래와 같은 경고 메시지가 표시됩니다.

{title="Command Line",lang="text"}
~~~~~~~~
Warning: Can only update a mounted or mounting component. This usually means you called setState, replaceState, or forceUpdate on an unmounted component. This is a no-op.
~~~~~~~~

이 경고 메시지를 없애기 위한 해결 방법은 컴포넌트가 마운트 해제될 때 API 요청을 중단하거나, 마운트 해제된 컴포넌트에서 `this.setState()` 메서드 호출을 방지하는 것입니다.
리액트 개발에서는 경고 메시지 없이 깨끗한 애플리케이션을 유지하는 것이 매우 중요합니다. promise에서 API 요청을 중단하게 처리하겠습니다. 아래 제가 소개하는 방법은 많은 개발자들이 따르고 있는 모범 사례입니다. 경고 메시지 내용을 해결할지, 그대로 내버려 둘지는 판단은 각자의 몫입니다. 이 책의 후반부에도 비슷한 경고 메시지가 나올 때마다 아래 방법으로 해결하면 됩니다.

해결 방법은 다음과 같습니다. 컴포넌트 생명주기 상태를 가리키는 클래스 필드인 `_isMounted`을 만듭니다. 초기값은 `false`입니다. 컴포넌트가 마운트 될 때 `true`로 변경되며, 컴포넌트가 마운트 해제될 때 다시 `false`로 변경됩니다. 이렇게 하면 컴포넌트 생명주기를 추적할 수 있습니다. 로컬 상태 아니기 때문에 `this.state`와 `this.setState()`와 무관합니다. 클래스 필드 값이 변경되어도 컴포넌트가 다시 렌더링 되지 않습니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
class App extends Component {
# leanpub-start-insert
  _isMounted = false;
# leanpub-end-insert

  constructor(props) {
    ...
  }

  ...

  componentDidMount() {
# leanpub-start-insert
    this._isMounted = true;
# leanpub-end-insert

    const { searchTerm } = this.state;
    this.setState({ searchKey: searchTerm });
    this.fetchSearchTopStories(searchTerm);
  }

# leanpub-start-insert
  componentWillUnmount() {
    this._isMounted = false;
  }
# leanpub-end-insert

  ...

}
~~~~~~~~

마지막으로 처음부터 요청을 중단하지 않고, 컴포넌트가 마운트 해제되었는지 확인하여 `this.setState()` 메서드 호출을 피하도록 수정하겠습니다. 이제 경고 메시지가 사라질 것입니다.

{title="src/App.js",lang=javascript}
~~~~~~~~
class App extends Component {

  ...

  fetchSearchTopStories(searchTerm, page = 0) {
    axios(`${PATH_BASE}${PATH_SEARCH}?${PARAM_SEARCH}${searchTerm}&${PARAM_PAGE}${page}&${PARAM_HPP}${DEFAULT_HPP}`)
# leanpub-start-insert
      .then(result => this._isMounted && this.setSearchTopStories(result.data))
      .catch(error => this._isMounted && this.setState({ error }));
# leanpub-end-insert
  }

  ...

}
~~~~~~~~

이번 절에서는 리액트에서 다른 라이브러리로 대체하는 방법을 배웠습니다. 자바스크립트에는 많은 라이브러리들이 있습니다. 개발 중 문제가 생기면, 여러 라이브러리를 탐색하고 적합한 솔루션을 도입해 해결합니다. 또한 마운트 해제된 컴포넌트에서 `this.setState()` 메서드 호출을 피하는 방법을 배웠습니다. 반대로 axios에서 API 요청 취소를 막는 방법도 있으니 직접 찾아보고 실습해보길 바랍니다. 

### 읽어보기

* [[저자 블로그] 왜 프레임워크가 중요한가](https://www.robinwieruch.de/why-frameworks-matter/)
* [[저자 리퍼지토리] 리액트 컴포넌트 대체 문법](https://github.com/rwieruch/react-alternative-class-component-syntax)

{pagebreak}

## 3.10 정리하면

3장에서는 외부 API를 사용하는 방법을 배웠습니다. 지금까지 학습한 내용을 정리해봅시다.

* 리액트
  * ES6 클래스 컴포넌트의 생명주기 메소드와 사용 사례
  * `componentDidMount()` 메서드에서 API 호출
  * 조건부 렌더링
  * 폼 이벤트
  * 에러 핸들링
* ES6
  * 템플릿 문자열 구성
  * 전개 연산자로 불변 데이터 구조 작성
  * 프로퍼티 이름 계산
* 일반
  * 해커 뉴스 API 인터렉션
  * 네이티브 브라우저 fetch API
  * 클라이언트 및 서버 내 검색 기능 구현
  * 페이지네이션 데이터 호출
  * 클라이언트 렌더링
  * fetch 대신 axios 도입
