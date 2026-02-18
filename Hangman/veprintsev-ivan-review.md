# Review на реализацию от [@veprintsev-ivan](https://github.com/veprintsev-ivan) проекта [Виселица](https://zhukovsd.github.io/java-backend-learning-course/projects/hangman/)

[Сама реализация](https://github.com/veprintsev-ivan/HangManGame)

## Реализация

#### Хорошо:

- Игра запускается и работает.
- Есть валидация введенных букв.
- Есть список ошибочных букв.
- Правильно отгаданные буквы открывают часть загаданного слова.
- Наличие главного меню для пользователя
- После конца игры перекидывает в главное меню
- После ошибки отображается следующий этап виселицы

#### Замечания:

- Как вариант улучшения можно рассмотреть увеличение кол-во слов.

## По коду

### HangmanApp

- Не стоит игнорировать замечания idea:

"Call to 'printStackTrace()' should probably be replaced with more robust logging".
Как одно из решений - выбросить исключение с содержательным сообщением:

```java
        try {
            wordProvider = new WordProvider();
        } catch (IllegalStateException e) {
            e.printStackTrace();
            return;
        }
// лучше
        try {
            wordProvider = new WordProvider();
        } catch (IllegalStateException e) {
            throw new IllegalStateException("WordProvider was not initialized", e);
        }

```

- На мой взгляд, странно пытаться корректно завершить программу при некорректной работе, лучше пробрасывать исключение.
  Исключение можно пробросить либо здесь же, либо еще в методе, который мы вызываем (о нем будет дальше в ревью).

```java
            String userAnswer = ConsoleUI.readUserAnswer();
            if (userAnswer == null) {
                ConsoleUI.printMessage("Input stream closed. Exiting game.");
                return;
            }
            
            // лучше
            String userAnswer = ConsoleUI.readUserAnswer();
            if (userAnswer == null) {
                throw new IllegalArgumentException("Input stream closed. Exiting game.");
            }

```
- "S", "E" - в данном случае магические числа, ведь несут бизнес-логику, а потому лучше заменить на константы:
```java
            switch (userAnswer) {
                case "S":
                    HangmanGame game = new HangmanGame(wordProvider);
                    game.start();
                    break;
                case "E":
                    ConsoleUI.printMessage("Bye!");
                    return;
                default:
                    ConsoleUI.printMessage("Unknown command. Press S to start or E to exit.");
            }
            
            // лучше
            switch (userAnswer) {
                case START_ANSWER:
                    HangmanGame game = new HangmanGame(wordProvider);
                    game.start();
                    break;
                case EXIT_ANSWER:
                    ConsoleUI.printMessage("Bye!");
                    return;
                default:
                    ConsoleUI.printMessage("Unknown command. Press S to start or E to exit.");
            }
```

- Стоит скрывать лишние детали реализации ради повышения читабельности кода:

```java
            if (userAnswer == null) {
                ConsoleUI.printMessage("Input stream closed. Exiting game.");
                return;
            }
            userAnswer = userAnswer.trim().toUpperCase();
// лучше
            if (isInputClosed(userAnswer)) {
                ConsoleUI.printMessage("Input stream closed. Exiting game.");
                return;
            }
            userAnswer = normalizeAnswer(userAnswer);
```

### ConsoleUI

- Т.к. это утилитарный класс, то ему стоит иметь закрытый конструктор и модификатор _final_:

```java
public class ConsoleUI {
    // пустой конструктор генерируется по умолчанию
}

// лучше
public final class ConsoleUI {
    private ConsoleUI() {
    }
    ...
}
```

- Класс нарушает SRP, т.к. исходя из своего названия класс должен содержать пользовательский интерфейс, но здесь он его
  не только содержит, но и сам его печатает, а еще принимает ответ от пользователя. Решением вижу декомпозицию на
  несколько классов по их ответственности, при этом мы все еще будем оставаться в процедурном стиле:

```java
public class ConsoleUI {
    String[] STAGES = {
            //1
            """
            _________
            |/      |
            |
            |
            |
            |
            |
          ===============
          """...}
          
    readUserAnswer()
    printMessage()
    printStartMenu()
    printGameState()
}
// лучше
public enum HangmanStages {
    FIRST_STAGE("""
              _________
              |/      |
              |
              |
              |
              |
              |
            ===============
            """),
            //...

    private final String image;

    HangmanStages(String image) {
        this.image = image;
    }

    public String getImage() {
        return image;
    }
}

public final class ConsoleRenderer {
    private ConsoleRenderer() {}

    printWinMessage(String secretWord)
    printLoseMessage(String secretWord)
    printCorrectGuessMessage()
    printIncorrectGuessMessage()
    //...
}

public final class ConsoleInputReader {
    private static final BufferedReader READER = new BufferedReader(new InputStreamReader(System.in));

    private ConsoleInputReader() {
    }

    readUserAnswer()
    // ...
}
```
- Никогда не возвращай _null_ - это сильно повышает вероятность _NPE_. Вместо этого используй Optional:

```java
            if (userAnswer == null) {
                return null;
            }
            
            // лучше
            if (userAnswer == null) {
                return Optional.empty();
            }
            // HangmanApp
            Optional<String> userAnswer = ConsoleUI.readUserAnswer();
            if (userAnswer.isEmpty()) {
                ConsoleUI.printMessage("Input stream closed. Exiting game.");
                return;
            }
```

- Проглатывать исключение очень спорное архитектурное решение, ведь ты теряешь _stacktrace_ и тем самым маскируешь
  реальную причину неполадки. Лучше пробросить исключение с содержательным сообщением или отдать обработку исключения
  верхним слоям:

```java
    public static String readUserAnswer() {
        try {
            String userAnswer = READER.readLine();
            if (userAnswer == null) {
                return null;
            }
            return userAnswer.trim();
        } catch (IOException e) {
            System.out.println("Input error. Please try again.");
            return null;
        }
    }
    // лучше
    public static Optional<String> readUserAnswer() {
        try {
            String userAnswer = READER.readLine();
            if (userAnswer == null) {
                return Optional.empty();
            }
            return Optional.of(userAnswer);
        } catch (IOException e) {
            throw new IllegalStateException("Input error. Please try again.", e);
        }
    }
```

- Также метод обманывает - помимо приема ответа юзера он так же его форматирует. Стоит либо поменять название или, что
  лучше, дать вызывающим решать, что делать с ответом юзера.
- `void printGameState(String maskedWord, int remainingAttempts, String wrongLetters)` - три аргумента у функции это уже
  достаточное кол-во, из-за которого можно ощутить дискомфорт в ее использовании. Обычно у такого кол-ва аргументов есть
  что-то общее и вместе они представляют собой какую-то сущность. В данном случае можем их вынести в новый класс
  _GameState_:

```java
    public static void printGameState(String maskedWord, int remainingAttempts, String wrongLetters) {
        System.out.print("\nWord: ");
        displayMaskedWord(maskedWord);
        displayHangmanStage(remainingAttempts);
        System.out.println("Remaining attempts: " + remainingAttempts);
        System.out.println("Wrong letters: " + wrongLetters);
        System.out.print("Enter one letter: ");

    }
// лучше
public record GameState(String maskedWord, int remainingAttempts, String wrongLetters) {
}
public class ConsoleUI {
    public static void printGameState(GameState gameState) {
        System.out.print("\nWord: ");
        displayMaskedWord(gameState.maskedWord());
        displayHangmanStage(gameState.remainingAttempts());
        System.out.println("Remaining attempts: " + gameState.remainingAttempts());
        System.out.println("Wrong letters: " + gameState.wrongLetters());
        System.out.print("Enter one letter: ");
    }
}
```

- В классе методы, отвечающие за отрисовку чего-либо не придерживаются единообразия: некоторые начинаются с _print_, а
  другие _display_.

### HangmanGame

- Поля с модификатором _final_ должны стоять выше обычных _instance fields_:
```java
public class HangmanGame {
    private static final int MAX_ATTEMPTS = 6;
    private final WordProvider wordProvider;
    private String secretWord;
    private String maskedWord;
    private int remainingAttempts;

    private final Set<Character> correctLetters = new LinkedHashSet<>();
    private final Set<Character> wrongLetters = new LinkedHashSet<>();
   
   // лучше
public class HangmanGame {
    private static final int MAX_ATTEMPTS = 6;
    private final WordProvider wordProvider;
    private final Set<Character> correctLetters = new LinkedHashSet<>();
    private final Set<Character> wrongLetters = new LinkedHashSet<>();
    private String secretWord;
    private String maskedWord;
    private int remainingAttempts;

```
- `String getLineWrongLetters(Set<Character> wrongLetters)`
    - В методе _StringBuilder_ использован по назначению - хорошо.
    - Но не совсем удачное название метода, на мой взгляд, _formatWrongLetters_ или _buildWrongLettersDisplay_ подошли
      бы лучше.
    - Метод затрагивает представление, в нашем случае, _wrongLetters_. Лучше отделять представление от бизнес-логики.
      Думаю этот метод хорошо подошел бы для _ConsoleUI_.
- Использование _LinkedHashSet_ для коллекций букв - идеально.
- Скрывай детали реализации, делай программу проще:

```java
while (remainingAttempts > 0 && !maskedWord.equals(secretWord))
// лучше
while (isGameNotOver())
...

userAnswer = userAnswer.trim();
// лучше
userAnswer = normalizeAnswer(userAnswer);    
...

char userLetter = Character.toUpperCase(userAnswer.charAt(0));
// лучше
char userLetter = normalizeLetter(userAnswer);
...

if (correctLetters.contains(userLetter) || wrongLetters.contains(userLetter))
// лучше
if (isLetterAlreadyUsed(userLetter))
...

if (secretWord.contains(String.valueOf(userLetter)))
// лучше
if (isCorrectLetter(userLetter))
...

if (maskedWord.equals(secretWord))
// лучше
if (hasPlayerWon())
```

- То, что писал выше относится и к этому вызову `ConsoleUI.readUserAnswer()`:

```java
            if (userAnswer == null) {
                ConsoleUI.printMessage("Input stream closed. Exiting game.");
                return;
            }
            // лучше
            if (userAnswer == null) {
                throw new IllegalStateException("Input stream closed. Exiting game.");
            }
```

- Так же, как в HangmanApp:

```java
            userAnswer = userAnswer.trim();
// лучше
            userAnswer = normalizeAnswer(userAnswer);
```

- Т.к. `isSingleEnglishLetter` это единственный метод, который валидирует, то его можно назвать проще `isAnswerValid`
  или `isInputValid`. Так же метод принимает в аргументы `String userAnswer`, хотя из оригинального названия логичнее
  подошел бы `char userLetter`.
- Такого рода валидацию так же можно вынести в отдельный класс `Validator`.

### WordProvider

- Не игнорируй подсказки _idea_ `Field 'words' may be 'final'`:

```java
private List<String> words;
// лучше
private final List<String> words;
```

- Загрузка слов в конструкторе - хорошо.
- При ошибке чтения файлов надо указать не только имя файла, который не обнаружен, но и абсолютный путь до него.
- В целом хороший класс, в котором четко ясна его ответственность.

### Общее

- Не забывай про подсказки от _idea_.
- В данный момент твой `root package` называется `main.java`, что не соответствует конвенциям, узнай как правильно их
  называть.
- Так же было бы полезно, на будущее, ознакомится с общепринятой Maven-структурой проекта.
- Проект написан в процедурном стиле в нескольких классах. Пример правильной объектно-ориентированной декомпозиции
  Виселицы на классы можешь посмотреть у Сергея в расширенных материалах. А также
  сравнить различия ООП и процедурным стилем:  
  Стрим Сергея [Крестики-нолики в процедурном стиле](https://www.youtube.com/watch?v=PPikj1qHxrA)  
  Стрим Алексея (tg:@Raketa4000az) [Крестики-нолики в ООП стиле](https://t.me/zhukovsd_it_chat/53243/187097)

## Итог
- Для процедурного стиля достойный проект, после рефакторинга можно смело идти дальше. Удачи!