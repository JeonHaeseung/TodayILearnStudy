# TIL - React로 QR Code Generater 만들어보기

## ReactPlay

[ReactPlay](https://reactplay.io/)는 작은 기능들을 하나씩 React로 만들어 볼 수 있도록 도와주는 사이트이다. 아주 간단한, 컴포넌트 수준의 기능부터, 체크리스트처럼 어느 정도 (초보자에겐) 복잡한 기능까지 만들어볼 수 있다. 리엑트를 처음 배워보는 김에 아주 간단하게, [QR 코드를 만들어주는 기능](https://reactplay.io/plays/murtuzaalisurti/qr-code-generator)을 따라해보기로 했다. 전체 코드는 아래와 같은데, 나는 정말 리엑트를 써본 것이 이게 처음이기 때문에 코드를 한줄 한줄 분석하는 것이 이번 TIL의 목표이다.

```javascript
import { useState } from "react";
import './QrCodeGenerator.css';
import QRCode from 'react-qr-code';
import * as htmlToImage from 'html-to-image';
import download from 'downloadjs';
import { MdDonutLarge } from 'react-icons/md';

function QrCodeGenerator(props) {
    // [현재 QR 코드에 표시할 값, 상태 업데이트] = 초기값
    // 이름은 마음대로 해도 되지만, 자리에 따라서 역할이 정해짐
    const [qrCodeValue, setQrCodeValue] = useState('https://reactplay.io');
    const [loading, setLoading] = useState(false);

    // 이벤트 핸들러, QrCodeGenerator 컴포넌트의 상태를 업데이트
    // e는 이벤트 객체, e.target은 이벤트가 발생한 HTML 요소, e.target.value는 이벤트가 발생한 요소의 현재 값
    function handleChange(e) {
        setQrCodeValue(e.target.value);
    }

    function handleDownload() {
        // 다운로드 중
        setLoading(true);
        // htmlToImage 라이브러리의 toJpeg 함수는 HTML 요소를 JPEG 이미지로 변환
        // JPEG로 변환할 HTML 요소를 선택
        // JPEG 이미지의 품질을 설정(1은 최고 품질)
        htmlToImage
            .toJpeg(document.querySelector('#qrContain'), {
                quality: 1
            })
            // Promise가 성공적으로 완료되었을 때 실행되는 콜백 함수
            // 즉, htmlToImage.toJpeg가 성공적으로 JPEG 이미지를 생성할 경우
            .then((dataUrl) => {
                // 이미지 파일 다운로드
                // 다운로드 완료
                download(dataUrl, 'qrcode.jpeg');
                setLoading(false);
            })
            // Promise가 실패했을 때 실행되는 콜백 함수를 정의
            // 로딩 상태를 false로 설정
            .catch(() => {
                setLoading(false);
            });
    }

    return (
        <>
            <div className="play-details">
                <div className="play-details-body">
                    {/* Your Code Starts Here */}
                    <div className="App">
                        <div id="qrContain" style={{ backgroundColor: 'white', width: 'fit-content' }}>
                            <QRCode
                                size={256}
                                value={
                                    qrCodeValue === undefined || qrCodeValue === ''
                                        ? 'https://reactplay.io'
                                        : qrCodeValue
                                }
                            />
                        </div>
                        <input
                            id="qrValue"
                            placeholder="Type something.."
                            type="text"
                            onChange={(e) => handleChange(e)}
                        />
                        <button id="download-btn" onClick={handleDownload}>
                            Download
                            {loading ? <MdDonutLarge /> : ''}
                        </button>
                    </div>
                    {/* Your Code Ends Here */}
                </div>
            </div>
        </>
    );
}

export default QrCodeGenerator;
```

## 라이브러리 사용하기

```javascript
import './QrCodeGenerator.css';
import QRCode from 'react-qr-code';
import * as htmlToImage from 'html-to-image';
import download from 'downloadjs';
import { MdDonutLarge } from 'react-icons/md';
```

FE의 세계에는 라이브러리가 정말 많다. 위의 코드는 이러한 라이브러리를 import해서 사용하는 것을 의미한다. React에서 라이브러리 관리는 패키지 매니저를 통해서 이루어진다. `build.gradle`에 직접 코드로 라이브러리를 명시했던 Spring Boot와는 달리, 리엑트에서는 터미널에서 패키지 매니저(나는 여기서 `pnpm`을 사용했다)를 통해 직접 설치 가능하다.

```bash
pnpm add react-qr-code
pnpm add html-to-image
pnpm add downloadjs
pnpm add react-icons
```

패키지 매니저는 라이브러리 설치 후, `package.json` 파일의 `dependencies` 섹션에 설치된 라이브러리를 명시한다.

```json
{
  "dependencies": {
    "downloadjs": "^1.4.7",
    "html-to-image": "^1.11.11",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-icons": "^5.3.0",
    "react-qr-code": "^2.0.15"
  }
}
```

## useState

```javascript
import { useState } from 'react';
const [state, setState] = useState(initialState);
```

`useState`는 상태 변수와 그 상태를 업데이트하는 함수를 선언하는 데 사용된다. 상태 변수는 컴포넌트의 상태를 저장하며, 상태 업데이트 함수는 상태를 변경하는 데 사용된다. 위의 코드에서 `state`는 현재 상태 값을, `setState`는 상태를 업데이트하는 함수를, `initialState`는 상태의 초기값을 의미한다. 참고로 이 변수명들은 자바의 생성자나 setter/getter처럼 반드시 이런 이름을 따라야 한다는 규칙이 있는 건 아니고, 내가 원하는 대로 설정 가능하다.

## handleChange

```javascript
function handleChange(e) {
    setQrCodeValue(e.target.value);
}
```

`handleChange`는 이벤트 핸들러로, `QrCodeGenerator` 컴포넌트의 상태를 업데이트한다. `e`는 이벤트 객체, `e.target`은 이벤트가 발생한 HTML 요소, `e.target.value`는 이벤트가 발생한 요소의 현재 값을 의미한다. handleChange 함수는 `QrCodeGenerator` 컴포넌트의 일부로 속해 있는데, 그 이유는 `qrCodeValue`의 상태를 업데이트하는 내부 상태 변경 함수이기 때문이다. HTML에서는 바로 이 `input` 필드가 변겅되는 이벤트가 발생하는 순간 `handleChange`가 호출되어서 `qrCodeValue`가 변경된다. 상태가 변경되면 React는 컴포넌트를 다시 렌더링한다. 이때 `<QRCode>` 컴포넌트의 `value`가 새로운 값으로 설정되므로, QR 코드가 새 값으로 업데이트된다.

```javascript
<div id="qrContain" style={{ backgroundColor: 'white', width: 'fit-content' }}>
    <QRCode
        size={256}
        value={
            qrCodeValue === undefined || qrCodeValue === ''
                ? 'https://reactplay.io'
                : qrCodeValue
        }
    />
</div>
<input
    id="qrValue"
    placeholder="Type something.."
    type="text"
    onChange={(e) => handleChange(e)}
/>
```

## handleDownload

```javascript
function handleDownload() {
    // 다운로드 중
    setLoading(true);
    // htmlToImage 라이브러리의 toJpeg 함수는 HTML 요소를 JPEG 이미지로 변환
    // JPEG로 변환할 HTML 요소를 선택
    // JPEG 이미지의 품질을 설정(1은 최고 품질)
    htmlToImage
        .toJpeg(document.querySelector('#qrContain'), {
            quality: 1
        })
        // Promise가 성공적으로 완료되었을 때 실행되는 콜백 함수
        // 즉, htmlToImage.toJpeg가 성공적으로 JPEG 이미지를 생성할 경우
        .then((dataUrl) => {
            // 이미지 파일 다운로드
            // 다운로드 완료
            download(dataUrl, 'qrcode.jpeg');
            setLoading(false);
        })
        // Promise가 실패했을 때 실행되는 콜백 함수를 정의
        // 로딩 상태를 false로 설정
        .catch(() => {
            setLoading(false);
        });
}
```

`setLoading`은 상태 업데이트 함수로, 상태 변수를 `loading`으로 설정한다. 이 함수가 호출되면 `loading`의 상태가 `true`로 설정된다. `useState`는 훅이고, 이 훅이 반환하는 상태 업데이트 함수가 `setLoading`으로 반환된다. 해당 함수는 아래 버튼에서 클릭 이벤트가 발생하는 순간 진행된다. 즉, 이미지 저장 버튼을 누르면 발생하는 일을 명시해놓은 함수이다. 참고로 promise는 JavaScript에서 비동기 작업의 완료 또는 실패를 나타내는 객체이다. 위의 코드에서 볼 수 있듯이, 비동기 작업이 완료된 후 실행할 코드는 then과 catch 메서드를 통해 정의된다.

```javascript
<button id="download-btn" onClick={handleDownload}>
    Download
    {loading ? <MdDonutLarge /> : ''}
</button>
```

## export

```javascript
export default QrCodeGenerator;
```

모듈을 내보내는 역할을 한다. 내보내고자 하는 값, 즉 `QrCodeGenerator` 컴포넌트를 모듈의 기본 내보내기로 설정한다.

## 완성된 화면

아래와 같이 URL을 입력하면 QR 코드를 생성해주고, 해당 이미지를 다운로드도 할 수 있는 기능을 만들었다. 아이폰 카메라 QR 코드 인식으로 실험해봤을 때 아주 정상적으로 작동한다!

<img style="width: 100%" alt="QR code generator image" src="https://github.com/user-attachments/assets/86937a35-7014-4ac8-af11-a85b06c70e67"/>

## 📜관련 자료
- https://reactplay.io/plays/murtuzaalisurti/qr-code-generator
