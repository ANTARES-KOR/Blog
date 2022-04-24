# MUI 코드 분석하기 1편 - Radio & RadioGroup

첫 블로그 게시물로 뭐가 좋을까 고민하다가 이번에 회사에서 새로운 프로젝트를 런칭하면서 UI 컴포넌트 라이브러리를 밑바닥부터 만들고 있는데, 참고도 하면서 정리하고자 하는 목적으로 MUI 라이브러리의 코드를 분석해 보기로 했다.

사실 미리 좀 뜯어보긴 했는데 진짜 너무너무너무 방대해서 어떻게 시작해야할 지 감이 안오긴하는데, 어떻게든 될거라고 생각하고 시작해보겠다.
그리고

**들어가기 전에**
우선 저도 이제 프론트엔드 개발을 시작한지 9개월밖에 안된 응애이기 때문에 잘못된게 있다면 화내지말고 차분하게 댓글로 지적해주시면 감사하겠습니다 :)

그리고 말투는 일단은 제가 반말로 글쓰는게 편해서 이렇게 쓰는데 나중에 바뀔지도..?

해당 포스트를 읽기 전 미리 필요한 사전지식을 정리해봤다. 모르겠다면 읽고 오면 좋을것 같다.

- [제어 컴포넌트와 비제어 컴포넌트](https://ko.reactjs.org/docs/uncontrolled-components.html)

## Radio Component 사용 예시 분석해보기

우선 [Mui Radio Component](https://mui.com/material-ui/react-radio-button/#main-content) 를 보면서 기본적으로 제공하는 API와 사용례부터 보자.

기본적으로 `RadioGroup` 컴포넌트 내부에서 `FormControlLabel`의 control 옵션에 대한 값으로 `Radio` 컴포넌트를 사용하고 있는것을 볼 수 있다.

갑자기 `FormControl`이랑 `FormControlLabel`, `FormLabel`이 등장해서 좀 당황스럽긴한데, 하나하나씩 다뤄보겠다.

```html
<FormControl>
  <FormLabel id="demo-radio-buttons-group-label">Gender</FormLabel>
  <RadioGroup
    aria-labelledby="demo-radio-buttons-group-label"
    defaultValue="female"
    name="radio-buttons-group"
  >
    <FormControlLabel value="female" control="{<Radio" />} label="Female" />
    <FormControlLabel value="male" control="{<Radio" />} label="Male" />
    <FormControlLabel value="other" control="{<Radio" />} label="Other" />
  </RadioGroup>
</FormControl>
```

또 다른 예시로는 Controlled 예시가 있는데 다음과 같다.

아까와의 차이점이라면 `RadioGroup` 에 `value`, `onChange` 속성이 부여되어있다는 점이다. 이렇게 사용하면 RadioGroup의 값을 Controlled 하게 다룰 수 있다는 예시로 생각하면 되겠다.

```Typescript
import * as React from 'react';
import Radio from '@mui/material/Radio';
import RadioGroup from '@mui/material/RadioGroup';
import FormControlLabel from '@mui/material/FormControlLabel';
import FormControl from '@mui/material/FormControl';
import FormLabel from '@mui/material/FormLabel';

export default function ControlledRadioButtonsGroup() {
  const [value, setValue] = React.useState('female');

  const handleChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    setValue((event.target as HTMLInputElement).value);
  };

  return (
    <FormControl>
      <FormLabel id="demo-controlled-radio-buttons-group">Gender</FormLabel>
      <RadioGroup
        aria-labelledby="demo-controlled-radio-buttons-group"
        name="controlled-radio-buttons-group"
        value={value}
        onChange={handleChange}
      >
        <FormControlLabel value="female" control={<Radio />} label="Female" />
        <FormControlLabel value="male" control={<Radio />} label="Male" />
      </RadioGroup>
    </FormControl>
  );
}
```

다음은 Standalone 방식이다. `RadioGroup`으로 감싸지 않고 `Radio` 혼자서 단독으로도 사용할 수 있다고 보여주는데, 한 번 같이 보자.

이 경우에는 `selectedValue`와 `onChange`는 주어지는 상태에서 `Radio` 컴포넌트를 사용하는 예시로 볼 수 있다.

Controlled Input이라고 보면 될거같다.

```Typescript
import * as React from 'react';
import Radio from '@mui/material/Radio';

export default function RadioButtons() {
  const [selectedValue, setSelectedValue] = React.useState('a');

  const handleChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    setSelectedValue(event.target.value);
  };

  return (
    <div>
      <Radio
        checked={selectedValue === 'a'}
        onChange={handleChange}
        value="a"
        name="radio-buttons"
        inputProps={{ 'aria-label': 'A' }}
      />
      <Radio
        checked={selectedValue === 'b'}
        onChange={handleChange}
        value="b"
        name="radio-buttons"
        inputProps={{ 'aria-label': 'B' }}
      />
    </div>
  );
}
```

그래서 대표적인 3가지 사용 예시를 뜯어봤는데, `RadioGroup`과 함께 사용할 때는 Controlled 와 Uncontrolled 방식 둘 다 사용 가능하고, `Radio`만 단독으로 사용할 때는 Controlled 방식으로만 사용 가능한 것으로 보인다.

자 그러면 Mui를 직접 뜯어보자!
코드를 직접 다 첨부하기에는 엄청나게 방대하기 때문에 여러분도 직접 [레포](https://github.com/mui/material-ui)를 포크해서 같이 따라오는 것을 추천한다.

## MUI 레포지토리 뜯어보기

처음 뜯어보면 난생 처음 보는 엄청난 코드에 되려 압도당하기 마련인데, 사실 우리에게 중요한 폴더는 몇개 안된다.

대부분은 배포, 문서화, 테스트 등등을 위한 폴더일 뿐이고 우리에게 중요한 폴더는 `packages` 폴더이다.

`/packages` 아래에 있는 폴더들이 우리가 npm에 [_@mui_](https://www.npmjs.com/search?q=%40mui) 를 검색하면 나오는 패키지들이다.

그 중 우리가 중점적으로 보게 될 폴더는 `/mui-material` 일 것이다.

### /src/Radio/Radio.js

```Javascript
const useUtilityClasses = (ownerState) => {
  ...
};

const RadioRoot = styled(SwitchBase, {
  ...
})(({ theme, ownerState }) => ({
  ...
}));

function areEqualValues(a, b) {
  ...
}

const defaultCheckedIcon = <RadioButtonIcon checked />;
const defaultIcon = <RadioButtonIcon />;

const Radio = React.forwardRef(function Radio(inProps, ref) {
  ...
  return (
    <RadioRoot
      type="radio"
      icon={React.cloneElement(icon, { fontSize: defaultIcon.props.fontSize ?? size })}
      checkedIcon={React.cloneElement(checkedIcon, {
        fontSize: defaultCheckedIcon.props.fontSize ?? size,
      })}
      ownerState={ownerState}
      classes={classes}
      name={name}
      checked={checked}
      onChange={onChange}
      ref={ref}
      {...other}
    />
  );
});

Radio.propTypes /* remove-proptypes */ = {
  ...
};

export default Radio;

```

처음 코드를 뜯어보면 어질어질하다. 앞으로 다른 녀석들도 그럴건데, 진짜 이걸 어떻게 만들었나 싶다. (사실 지금 괜히 이 글을 쓰기 시작했나 싶긴한데)
그래도 백준풀때의 정신으로 함수별로 Divide & Conquer 해나가다 보면 어떻게든 이해하는 경지가 되지 않을까? 하는 망상을 해본다.
그리고 사실 mui team 자체도 마소 출신 엔지니어, 등등등의 엄청난 분들이 만든 코드라서 지원하는 기능의 방대함에 비해서는 코드를 간결하게 짜놨을거라고 생각한다.

#### useUtilityclasses

일단 이름에서부터 나오는 향기는 utility class, 그러니깐 스타일링을 위한 class를 뽑아내는 custom Hook이지 않을까 하는 생각이 든다.

```Javascript
const useUtilityClasses = (ownerState) => {
  const { classes, color } = ownerState;

  const slots = {
    root: ['root', `color${capitalize(color)}`],
  };

  return {
    ...classes,
    ...composeClasses(slots, getRadioUtilityClass, classes),
  };
};
```

코드를 보면 `ownerState` 에서 `classes` 와 `color`를 뽑아낸 후,
이걸 가지고 `slot`을 생성한 후 이걸 또 `composeClasses` 함수에 다른 옵션들과 같이 집어넣어서 유틸리티 클래스 오브젝트를 만들어서 return 하는 것으로 보인다.

그럼 일단 slot에는 먼가 'root' 과 지정된 색상에 대한 정보가 들어가고, `getRadioUtilityClass`라는 함수?를 두번째 인자로 받고, ownerState로부터 분리한 classes property를 세번째 인자로 집어넣는다.
