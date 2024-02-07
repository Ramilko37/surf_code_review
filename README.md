## **Code-review**

**Общие замечания:** 

1. Иконки я бы положил в src/assets/icons;

  2. app стоит переименовать в store, useAppDispatch и useAppSelector я бы перенес в       store.ts;

 3. папку features имеет смысл перенести в store и переименовать в slices /или как вариант такой нейминг можно применить: redux => slices/..., store.ts;

 4. неплохо было бы объединить reducers в rootReducer;

1. Использовать css modules и scss.
2. Типизировать все initialStates в useState
3. Вынести обработчики кнопок в отдельные функции вместо повсеместного использования стрелочных функций
4. Необходимо отделить бизнес-логику от представления и вынести ее на верхний уровень приложения, например в app.tsx. Тем самым мы приблизимся к паттерну Container/Component. Компоненты должны рисовать представление и получать нужные данные в пропсах.  
5. Все модалки держать на верхнем уровне приложения и вынести их из компонентов.
6. В целом приложение небольшое и я бы не стал использовать виджеты. Но если оно будет расти, необходимо более сложные сущности вынести в виджеты, а в компонентах содержать только то, что может быть переиспользовано (ui-kit)
    1. Во всех стейтах нужно поработать над неймингом, например `selected` заменить на `isSelected` .

**categoriesSlice, tasksSlice**

```jsx
CategoriesSlice - неважный нейминг. 
Лучше interface Category {}, initialState CategoriesState = {value: Category[]}.

Стоит переименовать редюсеры  в addCategories, updateCAtegories etc
Аналогичные замечания для tasksSlice

Нейминг интерфейсов, редюсеров. Какие-то пейлоады получают <T>, какие-то нет. 
Нужно типизировать все payloads.

Рекомендую импортировать PayloadAction из "@reduxjs/toolkit"

 И с новой типизацией initialState можно будет 
упростить например добавление и удаление итемов в таком ключе - 
addItem (state: TasksState, action: PayloadAction<Task>) => 
state.value = [...state.value, action.payload]

removeItem (state: TasksState, action: PayloadAction<string>)
state.value = state.value.filter(x => x.id != action.payload) 

Что касается updated reducer, лучше применить подход, который декларирует Redux
при работе с редюсерами, a именно - "They are not allowed to modify the existing
state. Instead, they must make immutable updates, by copying the existing state
and making changes to the copied values. 
Как пример - tasksUpdated: (state, action) => { 
const { id, name, description, category } = action.payload;
return state.map((task) => { 
     if (task.id === id) {
      return {      ...task,      name,      description,      category,      };
      }      return task; });  },

SelectAll я бы переименовал в GetAll.
```

**Header**

```jsx
const { pathname } = useLocation(),
    isCategories = pathname.includes("categories"),
    [createModalActive, setCreateModalActive] = useState(false);

Плохая практика, ухудшает читаемость кода. Лучше - 
const { pathname } = useLocation(),
const isCategories: boolean = pathname.includes("categories"),
const [createModalActive, setCreateModalActive] = useState<boolean>(false);

const { pathname } = useLocation() если уж так хочется опираться на строку из урла
можно вынести наверх и прокидывать детям в виде пропса, а не вызывать повсеместно

onClick={() => {setCreateModalActive(true)}}  - вынести в функцию обработчик
и вместе с модалкой вынести их в родителя. Header получит handler в пропсах
и отработает.
```

**Lists**

```jsx
Здесь можно порефакторить и сократить количество компонентов. Я бы объединил 
Tasks и Categories в один компонент. Например - RenderItemsComponent.
Они одинаковые и отличаются только входными данными. RenderItemsComponent получит в
пропсах categories или tasks и отрисует <ListItem />

Можно задать enum DataType = {
 Categories,
 Tasks
} и оперировать им.

Получившийся компонент было бы неплохо обернуть в React.memo, чтобы избежать лишних
ререндеров. Примерно так - 

enum DataType {
  Categories,
  Tasks,
}

interface IRenderItemsComponent {
  items: (DataType.Categories | DataType.Tasks)[];
}

export const RenderItemsComponent = React.memo( ({items}: IRenderItemsComponent) => {

  return (
    <ul>
      {items.map((item: (DataType.Categories | DataType.Tasks)) => (
        <ListItem key={item.id} item={item} />
      ))}
    </ul>
  );
},
    (prevProps, nextProps) => prevProps.items === nextProps.items
)

Кстати не очень понятно почему в app.css обнаружились стили /* Lists */

**ListItem** 

const categories = useSelector(selectAllCategories),
      [editModalActive, setEditModalActive] = useState(false)
  let [removeModalActive, setRemoveModalActive] = useState(false);

****Аналогичное выше замечание + let в useState это плохо, поскольку мы не хотим 
мутировать initialState.

interface ListItemProps {
  item: {
    id: string;
    name: string;
    description: string;
    category?: string;
  };
} по хорошему под категории и таски нам нужен общий интерфейс в духе 

export interface IItem extends Category, Task {}

onClick={() => {
              removeModalActive = true;
            }} ошибка из-за которой не работет удаление. onClick={() => {
              setRemoveModalActive( true)
            }} ну и в хендлер.

<img src={edit} alt="edit" /> лучше называть иконки более очевидно - editIcon.

<ModalEditItem
          item={item}
          active={editModalActive}
          setActive={setEditModalActive}
        />
        <ModalRemoveItem
          item={item}
          active={removeModalActive}
          setActive={setRemoveModalActive}
        /> - модалки вынести наверх.
```

**Modal**

```jsx
Модалку мы тоже порефакторим. Сейчас здесь все в кучу, есть идея сделать компонент,
который занимается отрисовкой модалки и вызывать его с разными пропсами, вместо того
чтобы плодить разные компоненты одной по сути модалки (ModalRemoveItem,
ModalCreateItem, etc)

Я бы назвал его <RenderModalComponent
									 handlers={handlers}
									 headerContent={}
									 bodyContent={}
									 footerContent={}
									 ...props
 /> 
Внутри RenderModalComponent возвращает: 

const ModalHeader: ComponentWithAs<"header", someProps> =
 <header>{...content, props, etc}</header> 
// кстати не уверен что header здесь семантичен. мб стоит использовать
// div?  источник: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/header

const ModalContent: ComponentWithAs<"div", someProps> = 
 в зависимости от типа модалки и пропсов будет рисовать наполнение модалки,

Здесь я бы поработал над контентом. Если посмотреть на модалки edit, create -
 их можно поделить по смыслу на две категории - withDropdown и !withDropdown.
Соотв-но ModalContent нужно собирать исходя из входных данных. Можно например 
возвращать его вариации в switch case:

enum ModalType = {
	EditTask,
	CreateTask,
	EditCategory,
	CreateCategory,
	RemoveItem
}

и далее в зависимости от входных данных рисовать 
const ModalContent: ComponentWithAs<"div", someProps> = 
	switch(someProps.modalType) {
		case (ModalType.EditTask): 
			return (<></>),
		case (ModalType.CreateTask):
			return (<></>),
	} ...
	
const ModalFooter: ComponentWithAs<"div", someProps> = по аналогии. Он очень
простой, пробрасываем нужные кнопки и handlers. И безусловно необходимо избавиться
от конструкций вроде этой: onSubmit={
          name
            ? () => {
                dispatch(
                  isCategories
                    ? categoriesAdded({ name, description })
                    : tasksAdded({
                        name,
                        description,
                        category: setSelected, // Кстати здесь теряется категория 
																							 // при создании новой таски
                      })                       // Должно быть setSelected(selected)           
                );
                clearState();
                setActive(false);
              }
            : () => {}
        }
Вложенные тернарники это плохо. Нужно вынести в обработчик и передавать
 аргументами тип payload, category: setSelected

export const ModalText: React.FC<ModalTextProps> = ({ text }) => {
  return <p className="modal__content-text">{text}</p>;
}; - Разбиение на компоненты это хорошо, но здесь можно просто положить 
<p className="modal__content-text">{text}</p> в ModalContent. 
Тем более, что мы его используем только в одном месте.

**ModalBtn** - 

const btnClass =
    type === "primary"
      ? size === "large"
        ? "modalbtn primary large"
        : "modalbtn primary"
      : "modalbtn"; Плохо читается, добавится еще пара вариантов и станет совсем нечитаемо.
Лучше использовать что-то такое:
 const classMap = {
  primary: {
    large: "modalbtn primary large",
    default: "modalbtn primary",
  },
  default: "modalbtn",
}; нечто похожее применяется в ui библиотеках, например ChakraUI. 
Можно завести объект стилей кнопки, описать дефолтные стили и все необходимые
варианты стилизации и удобно их использовать. 

**ModalRow** просто обертка .modal__content_row {
  display: flex;
  justify-content: space-between;
}.   Это можно хранить внутри ModalContent и не выносить в отдельный компонент.

В результате у нас получится компонент модалки который будет лежать в папке
Modal/RenderModalComponent, styles.module.scss.  ModalComponents/ ModalBtn, 
ModalInput, ModalTextArea, ModalDropDown

Мы избавились от лишнего кода и сделали модалку удобной в переиспользовании.
Сперва я хотел применить здесь паттерн CompoundComponents.
Но решил, что это излишне да и общего контекста у модалок нет.

 Все инстансы модалки вызываются в app.tsx по условиям.

```

![Screenshot 2023-09-29 at 10.43.28.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/e0a944f0-d321-4638-9c2a-81f99b5c53aa/785edc9f-2031-44dc-a83d-5062d5068f15/Screenshot_2023-09-29_at_10.43.28.png)

Это место нужно пофиксить.

**index.css**

`code {  font-family: source-code-pro, Menlo, Monaco, Consolas, "Courier New",    monospace;}` У нас вроде нигде этот элемент не используется.
