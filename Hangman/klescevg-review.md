# Review на реализацию от [@klescevg](https://github.com/klescevg) проекта [Виселица](https://zhukovsd.github.io/java-backend-learning-course/projects/hangman/)
[Сама реализация](https://github.com/klescevg/hangman-procedural)

## Реализация
#### Хорошо:
- Игра запускается и работает.
- Есть валидация введенных букв.
- Есть список ошибочных букв.
- Правильно отгаданные буквы открывают часть загаданного слова.

#### Замечания:
- Нет отображения виселицы - нарушение ТЗ.
- При старте приложения не предлагает начать новую игру или выйти из приложения - нарушение ТЗ.
- По завершению игры не предлагается начать новую игру или выйти из приложения - нарушение ТЗ.

## По коду
### HangmanGame
- По общепринятым конвенциям поля-константы всегда идут выше остальных:
```java
// сейчас
public class HangmanGame {
    private static String word;
    private static final Set<Character> correctLetters = new LinkedHashSet<>();
    private static final Set<Character> incorrectLetters = new LinkedHashSet<>();

// лучше
    public class HangmanGame {
        private static final Set<Character> correctLetters = new LinkedHashSet<>();
        private static final Set<Character> incorrectLetters = new LinkedHashSet<>();
        private static String word;
```
- (опциональное улучшение) Для удобочитабельности можно заменить на константу с содержательным именем:
```java
// сейчас
    public static void playGame(List<String> words) {
        System.out.println("Welcome to new game!");

// лучше
    private static final String WELCOME_MESSAGE = "Welcome to new game!";
    
    public static void playGame(List<String> words) {
    
        System.out.println(WELCOME_MESSAGE);
```
Это также применимо ко всем сообщениям юзеру в проекте.
- Для улучшения читабельности программы можно располагать функции в порядке понижения абстракции, пример:
```java
// сейчас
private static char getInputLetter()

private static char getNewLetter() // функция getInputLetter вызывается в getNewLetter

// лучше

private static char getNewLetter() // сначала обозреватель видит getNewLetter, в которой вызывается getInputLetter, и опускаясь ниже, видит сам getInputLetter

private static char getInputLetter()
```
- _word_, _correctLetters_, _incorrectLetters_ - состояние игры как глобальные static поля, что не позволяет без дополнительных методов реализовать начало новой игры после уже законченной. Предлагаю держать состоянии локально, а не в static:
```java
// сейчас
public class HangmanGame {
    private static final Set<Character> correctLetters = new LinkedHashSet<>();
    private static final Set<Character> incorrectLetters = new LinkedHashSet<>();
    private static String word;

    public static void playGame(List<String> words) {
        System.out.println("Welcome to new game!");

        word = getRandomWord(words);

        while (!isGameOver()) {
            showRoundResults();
            playRound();
        }

        showGameResults();
    }
    
// лучше
    public static void playGame(List<String> words) {
        System.out.println("Welcome to new game!");

        String word = getRandomWord(words);
        Set<Character> correctLetters = new LinkedHashSet<>();
        Set<Character> incorrectLetters = new LinkedHashSet<>();

        while (!isGameOver(word, correctLetters, incorrectLetters)) {
            showRoundResults(word, correctLetters, incorrectLetters);
            playRound(word, correctLetters, incorrectLetters);
        }

        showGameResults(word, correctLetters, incorrectLetters);
    }
// в таком случае для удобства может добавить класс GameState, который будет содержать наше состояние, и проект все еще останется в процедурном стиле

public record GameState(String word, Set<Character> correctLetters, Set<Character> incorrectLetters) {
}
public static void playGame(List<String> words) {
    System.out.println("Welcome to new game!");

    String word = getRandomWord(words);
    Set<Character> correctLetters = new LinkedHashSet<>();
    Set<Character> incorrectLetters = new LinkedHashSet<>();

    GameState gameState = new GameState(word, correctLetters, incorrectLetters);

    while (!isGameOver(gameState)) {
        showRoundResults(gameState);
        playRound(gameState);
    }

    showGameResults(gameState);
}
```
### DictionaryReader

- `Scanner dictionaryScanner = new Scanner(dictionaryPath, UTF_8);` - сканнер нигде не закрывается, утечка ресурсов. Решение - использовать _try-with-resources_:
```java
// сейчас
try {
            Scanner dictionaryScanner = new Scanner(dictionaryPath, UTF_8);
            ...
// лучше
try (Scanner dictionaryScanner = new Scanner(dictionaryPath, UTF_8)) {
    ..
```
- `dictionaryScanner.hasNext()` - метод проверяет наличие токена, а не строки. Логичнее было бы использовать `dictionaryScanner.hasNextLine()`
- `System.exit(1);` - архитектурная ошибка. Класс не должен решать, когда надо завершать _JVM_. Решение - пробросить _unchecked_ исключение:
```java
// сейчас
catch (IOException e) {
            System.err.println("Cannot read dictionary file: ");
            System.err.println(dictionaryPath.toAbsolutePath());
            System.exit(1);
        }

// лучше
catch (IOException e) {
            throw new IllegalStateException(
                    "Cannot read dictionary file: " + dictionaryPath.toAbsolutePath(), e);
        }
```
### Main
- Просто вызывает метод _HangmanGame_ - хорошо.

### Итог
- Неплохой проект, в котором реализована механика "Виселицы", пусть и без визуала. Виден опыт в программировании, после рефакторинга некоторых пунктов можно смело идти дальше. Удачи!