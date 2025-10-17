## 트러블슈팅: React(Vite)와 FastAPI(WebRTC) 연동 테스트

* **작성자:** [고동현](https://github.com/rhehdgus8831)

-----

### 1. 문제 현상 (Problem)

> React 클라이언트와 FastAPI 서버 간의 통신이 실패하여 다양한 네트워크 오류가 발생했습니다.

* **문제 1**: React(Vite) 클라이언트에서 FastAPI 서버로 API 요청 시 브라우저 콘솔에 `TypeError: Failed to fetch`, `CORS policy` 위반, `404 Not Found` 오류가 복합적으로 발생했습니다.
* **문제 2**: FastAPI 서버는 로컬에서 정상적으로 실행되고 모델 로딩까지 완료되었으나, 클라이언트의 요청을 제대로 처리하지 못했습니다.
* **문제 3**: Vite 프록시와 FastAPI의 CORS 설정을 적용했음에도 WebSocket 연결이 불안정하거나 전혀 이루어지지 않았습니다.

<br>

**[문제 상황 스크린샷 및 로그]**

```bash
# 인코딩 문제 로그
ERROR: Exception:
Traceback (most recent call last):
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\cli\base_command.py", line 160, in exc_logging_wrapper
    status = run_func(*args)
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\cli\req_command.py", line 247, in wrapper
    return func(self, options, args)
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\commands\install.py", line 363, in run
    reqs = self.get_requirements(args, options, finder, session)
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\cli\req_command.py", line 433, in get_requirements
    for parsed_req in parse_requirements(
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\req\req_file.py", line 145, in parse_requirements
    for parsed_line in parser.parse(filename, constraint):
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\req\req_file.py", line 327, in parse
    yield from self._parse_and_recurse(filename, constraint)
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\req\req_file.py", line 332, in _parse_and_recurse
    for line in self._parse_file(filename, constraint):
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\req\req_file.py", line 363, in _parse_file
    _, content = get_file_content(filename, self._session)
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\req\req_file.py", line 541, in get_file_content
    content = auto_decode(f.read())
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\utils\encoding.py", line 34, in auto_decode
    return data.decode(
UnicodeDecodeError: 'cp949' codec can't decode byte 0xec in position 27: illegal multibyte sequence
```

```bash
# C++ 빌드 도구 오류 로그
Building wheels for collected packages: av
  Building wheel for av (pyproject.toml) ... error
  error: subprocess-exited-with-error
  × Building wheel for av (pyproject.toml) did not run successfully.
  │ exit code: 1
  ╰─> [98 lines of output]
      C:\Users\user\AppData\Local\Temp\pip-build-env-w31n3ni9\overlay\Lib\site-packages\setuptools\_distutils\dist.py:289: UserWarning: Unknown distribution option: 'test_suite'
        warnings.warn(msg)
      C:\Users\user\AppData\Local\Temp\pip-build-env-w31n3ni9\overlay\Lib\site-packages\setuptools\dist.py:759: SetuptoolsDeprecationWarning: License classifiers are deprecated.
      !!
      running bdist_wheel
      running build
      running build_py
      running build_ext
      building 'av.buffer' extension
      error: Microsoft Visual C++ 14.0 or greater is required. Get it with "Microsoft C++ Build Tools": https://visualstudio.microsoft.com/visual-cpp-build-tools/
      [end of output]
  note: This error originates from a subprocess, and is likely not a problem with pip.
  ERROR: Failed building wheel for av
Failed to build av
ERROR: Could not build wheels for av, which is required to install pyproject.toml-based projects
```

-----

### 2. 원인 분석 (Analysis)

> 클라이언트와 서버 간의 통신 방식 불일치와 환경 설정 미비로 인해 연결이 실패했습니다.

* **원인 1: 환경 설정 미비**

  * Windows 환경에서 인코딩 문제(`UnicodeDecodeError: 'cp949'`)와 C++ 빌드 도구 부재(`Microsoft Visual C++ 14.0 is required`)로 인해 서버의 의존성 라이브러리가 정상적으로 설치되지 않았습니다.

* **원인 2: CORS 및 프록시 설정 오류**

  * React(`http://localhost:5173`)와 FastAPI(`http://127.0.0.1:8000`)가 서로 다른 출처를 가져 CORS 정책 위반이 발생했으며, Vite의 프록시 설정에서 WebSocket(`ws: true`) 옵션 누락으로 연결이 불완전했습니다.

* **원인 3: 아키텍처 불일치**

  * 클라이언트는 `fetch` API를 통해 `/predict_video`로 요청을 보냈으나, 서버는 WebSocket과 WebRTC DataChannel 기반의 실시간 스트리밍을 기대하여 `404 Not Found` 오류가 발생했습니다.

-----

### 3. 해결 방안 (Solution)

> 서버와 클라이언트의 환경을 안정화하고 통신 아키텍처를 WebSocket 기반으로 통일하여 문제를 해결했습니다.

* **[해결 방안 1]: 서버 환경 안정화**

  * 인코딩 문제는 PowerShell에서 `$env:PYTHONUTF8=1`을 설정하여 `requirements.txt`를 UTF-8로 읽도록 했으며, C++ 빌드 도구 문제는 [Unofficial Windows Binaries for Python Extension Packages](https://www.lfd.uci.edu/~gohlke/pythonlibs/)에서 미리 컴파일된 `av` 라이브러리(`.whl` 파일)를 다운로드해 설치했습니다.

* **[해결 방안 2]: 프론트엔드 환경 안정화**

  * `janus.js`와 `adapter.js`의 레거시 라이브러리 문제를 해결하기 위해 초기에는 `index.html`에 `<script>` 태그로 전역 로드했으며, 최종적으로 `GameRoom.jsx`에서 `useEffect`를 사용한 동적 스크립트 로딩으로 성능을 최적화했습니다:
    ```jsx
    // GameRoom.jsx
    const [isJanusLoaded, setIsJanusLoaded] = useState(false);
    useEffect(() => {
        const loadScript = (url) => {
            return new Promise((resolve, reject) => {
                if (document.querySelector(`script[src="${url}"]`)) {
                    return resolve();
                }
                const script = document.createElement('script');
                script.src = url;
                script.async = true;
                script.onload = () => resolve();
                script.onerror = (err) => reject(err);
                document.head.appendChild(script);
            });
        };
        loadScript('https://cdnjs.cloudflare.com/ajax/libs/webrtc-adapter/8.2.3/adapter.min.js')
            .then(() => loadScript('/janus.js'))
            .then(() => {
                if (window.Janus) {
                    setIsJanusLoaded(true);
                } else {
                    throw new Error("Janus object not found.");
                }
            })
            .catch(error => console.error("Script loading error:", error));
    }, []);
    useEffect(() => {
        if (isJanusLoaded) {
            window.Janus.init({ /* Janus 초기화 로직 */ });
        }
    }, [isJanusLoaded]);
    ```

* **[해결 방안 3]: 통신 아키텍처 통일 및 프록시 설정**

  * 클라이언트 로직을 WebSocket과 WebRTC DataChannel 기반으로 수정하여 서버와 동일한 프로토콜을 사용하도록 했으며, Vite 프록시 설정에 `ws: true`를 추가해 WebSocket 요청을 정상적으로 전달했습니다:
    ```javascript
    // vite.config.js
    server: {
      proxy: {
        '/api': {
          target: 'http://127.0.0.1:8000',
          changeOrigin: true,
          ws: true,
          rewrite: (path) => path.replace(/^\/api/, '')
        }
      }
    }
    ```

-----

### 4. 교훈 (Lessons Learned)

> 풀스택 개발에서 시스템 전체의 데이터 흐름과 통신 방식을 명확히 설계하는 것이 중요하다는 것을 깨달았습니다.

* **[교훈 1]**: 클라이언트와 서버의 통신 프로토콜을 사전에 명확히 정의하지 않으면 아키텍처 불일치로 인해 불필요한 디버깅 시간이 소요됩니다.
* **[교훈 2]**: CORS, 프록시, WebSocket 등 네트워크 관련 설정은 이론적 이해를 넘어 실제 로그 추적과 테스트를 통해 문제를 해결해야 합니다.
* **[교훈 3]**: 레거시 라이브러리 연동 시 동적 로딩과 같은 최적화 기법을 적용하면 성능과 유지보수성을 동시에 개선할 수 있습니다.