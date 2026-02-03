## Frontend тестовое задание

### Формат

- Один файл/документ
- Можно писать псевдокод
- Можно использовать AI
- Ничего реально запускать не нужно
- Важны решения, а не объем
## Тест. QuestionCard + устойчивый UI (frontend мышление)
### Контекст (2 строки)
Есть QuestionCard.
Контент вопроса приходит как JSON (TipTap), формулы через KaTeX.
Пользователь кликает быстро, API иногда косячит.

---
### Часть 1. Архитектура компонента
```
QuestionCard
├── QuestionContent (tittap + katex)   ← useQuestionStore(selectContent)
├── AnswerOptions (useQuestionStore(selectOptions) массив вариантов), с разным типом 'radio' | 'checkbox') 
│   └── AnswerRadioButton || AnswerCheckbox (value, type, state: idle|selected|correct|incorrect)
├── Actions ← useQuestionStore(selectCanCheck)
│   └── CheckButton (isAnswerExist: useQuestionStore(selectIsAnswerExist)) / NextButton (conditional: если есть еще)
├── Explanation (conditional)
│   └── ← useQuestionStore(selectShowExplanation)
```

| Состояние        | Где хранится  | Причина                                         | Описание                                                                                    |
| ---------------- | ------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `selectedAnswer` | questionStore | Вопрос-специфично, сбрасывается при setQuestion | контент вопроса                                                                             |
| `isChecked`      | questionStore | сбрасывается при setQuestion                    | Состояние на определения, был ли проверен ответ                                             |
| `questionId`     | questionStore |                                                 | уникальный индетификатор                                                                    |
| `isChecking`     | questionStore | сбрасывается при setQuestion                    | флаг для определения состояние процесса проверки ответа, предотвратит множественные запросы |

При изменении `questionId` (пропс) вызывается `setQuestion(id)`:
- `selectedAnswer` → null
- `isChecked` → false
- `isChecking` → false (и abort активного запроса)
`explanation` и `correctAnswer` приходят с новыми пропсами.

### Часть 2. Псевдокод логики

```ts
---Store---
create<QuestionState>((set, get) => ({
  questionId: null,
  selectedAnswer: null,
  isChecked: false,
  isChecking: false,
  setQuestion: (id) => {
    set({ questionId: id, selectedAnswer: null, isChecked: false, isChecking: false })
  },
  selectAnswer: (value) => set((state) =>
    !s.isChecked ? { selectedAnswer: value } : {}
  ),
  checkAnswer: async (questionId, api) => {
    const { selectedAnswer, isChecked, isChecking } = get()
    if (!selectedAnswer || isChecked || isChecking) return
    set({ isChecking: true })
    try {
      const result = await api.checkAnswer(questionId, selectedAnswer)
      set({ isChecked: true, correctAnswer: result.correctAnswer, explanation: result.explanation })
    } catch (e) {
      someErrorAlert(e)
    } finally {
      if (get().questionId === questionId) set({ isChecking: false, abortController: null })
    }
  },
}))
```
```
---Selectors---
selectContent = store => selectContent.selectedAnswer.contenet

selectOptions = store => selectContent.selectedAnswer.options

selectCanCheck = store => store.selectedAnswer !== null && !store.isChecked && !store.isChecking

selectExplanationPlaceholder = store => store.isChecked && !store.isDemoMode ? selectedAnswer.placeholder : null
```
```
---Disabled состояния---

CheckButton.disabled = !canCheck

CheckButton.loading = isChecking

AnswerOptions.disabled = isChecked

NextButton.visible = isChecked
```

### Часть 3. Edge cases и UX

### 1. Explanation отсутствует
- Показать блок: «Пояснение к этому вопросу недоступно» или нейтральное сообщение
- Не показывать пустой блок — только если `explanation` есть или показать placeholder
- Альтернатива: скрыть блок Explanation полностью, если `!explanation`
### 2. В stem только формулы
- `QuestionContent` рендерит пустой контейнер с формулами
- Минимальная высота/отступы, чтобы формулы не «слипались» с границами
- Убедиться, что `TipTapRenderer` не скрывает контент при отсутствии текста
### 3. Очень длинный текст в stem
- Скролл внутри `QuestionContent`или скролл всей карточки 
### 4. KaTeX упал с ошибкой
- Сообщение: «Не удалось отобразить формулу»
- Не ломать все — рендерить остальной контент, проблемную формулу заменить placeholder'ом
### 5. Пользователь меняет ответ после check
- **Не получится с данной реализацией стора** — `AnswerOptions` disabled/read-only при `isChecked`
### 6. Demo режим
- **Explanation** — скрыт или заблюрен
- Текст: «Подробное объяснение доступно в полной версии» / «Разблокируйте доступ для просмотра объяснений»