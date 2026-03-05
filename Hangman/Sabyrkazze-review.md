# Review на реализацию от [@Sabyrkazze](https://github.com/Sabyrkazze) проекта [Виселица](https://zhukovsd.github.io/java-backend-learning-course/projects/hangman/)

[Сама реализация](https://github.com/Sabyrkazze/HangManGame)

## Реализация

#### Хорошо:

- Игра запускается и работает.
- При старте предлагает начать новую игру или выйти.
- По завершению вместе с результатами предлагает еще раз сыграть или выйти.
- Нельзя дважды ввести одну и ту же букву.
- Есть возможность выбрать сложность.

#### Замечания:

- Не выводится в консоль счетчик ошибок - нарушение _ТЗ_.
- Можно ввести букву из любого алфавита.
- Хотелось бы сообщение при ошибке валидации инпута юзера во время игры.
- На мой вкус: запятые у отображения _hidden word_ немного мешают.

## По коду

### Main

- Нарушение _Java_ конвенции - `static final` константы находятся ниже _instance fields_.
- Отсутствие модификаторов доступа на _instance fields_ - нарушение инкапсуляции. Также модификатор `final` не был бы
  здесь лишним, т.к. поля не изменяются по ходу программы:

```java
public class Main {

    Scanner scanner = new Scanner(System.in);
    PrintUtils printUtils = new PrintUtils();
    Validation validation = new Validation();
    ReadingUtils reading = new ReadingUtils();
    TextProcessor textProcessor = new TextProcessor();
    FileReadingUtils fileReading = new FileReadingUtils();

    private static final int QUIT = 0;
    private static final int PLAY = 1;

// лучше
public class Main {

    private static final int QUIT = 0;
    private static final int PLAY = 1;
    private final Scanner scanner = new Scanner(System.in);
    private final PrintUtils printUtils = new PrintUtils();
    private final Validation validation = new Validation();
    private final ReadingUtils reading = new ReadingUtils();
    private final TextProcessor textProcessor = new TextProcessor();
    private final FileReadingUtils fileReading = new FileReadingUtils();
```

- Каждое поле объявляется через _field initialization_, что допустимо, но было бы гибче, если бы зависимости
  инжектировались через конструктор.
- Нарушение _SRP_ - класс, помимо логики игры, сам контролирует свой жизненный цикл. Стоит вынести логику игры в
  отдельный класс. Также, чтобы освобождать ресурсы `Scanner`, можно использовать его с `try-with-resources` После всех
  изменений `main` выглядел бы так:

```java
    public static void main(String[] args) {
        Main main = new Main();
        main.start();
    }
    
// лучше

    public static void main(String[] args) {
        try (Scanner scanner = new Scanner(System.in);) {
            HangmanGame game = new HangmanGame( // инжект зависимостей через конструктор
                    new PrintUtils(),
                    new Validation(),
                    new ReadingUtils(scanner),
                    new TextProcessor(),
                    new FileReadingUtils()
            );
            game.start();
        }
    }
}
```

`public void start()`:

- Так как метод вызывается только внутри своего класс, то стоит неправильный модификатор доступа: `public` -> `private`.
    - Метод имеет существенные признаки божественного метода, а именно его длина и смешение ответственностей. Стоит
      воспользоваться вспомогательными методами и сделать код более удобочитабельным:

      ```java
          public void start(){
  
            while(true){
                int answer;
                try {
                    printUtils.printStartOrExitMenu();
                    answer = reading.readChoice(scanner);
                } catch (InputMismatchException e) {
                    printUtils.printNotValidChoice();
                    scanner.nextLine();
                    continue;
                }
  
                if(answer == QUIT){
                    System.out.println("Good Bye!");
                    break;
                }
                else if (answer != PLAY){
                    printUtils.printNotValidChoice();
                    continue;
                }
                //Game Starts
                StringBuilder wrongLetters = new StringBuilder();
                printUtils.printLetsStart();
                int difficulty;
                ...
      // лучше
          public void start() {
  
            while (true) {
                printUtils.printStartOrExitMenu();
                int answer = getChoice(scanner); // инкапсулировали чтение инпута игрока
  
                if (answer == PLAY) { // логика упрощена, избавились от continue
                    startRound();
                } else if (answer == QUIT) {
                    System.out.println("Good Bye!");
                    break;
                } else {
                printUtils.printNotValidChoice();
                }
              
            }
        }
    
      private void startRound() {
            StringBuilder wrongLetters = new StringBuilder();
            printUtils.printLetsStart();
            ...
      ```
- Анти паттерн _"exceptions as control flow"_. Как один из вариантов решения:

```java
    public void start(){

        while(true){
            int answer;
            try {
                printUtils.printStartOrExitMenu();
                answer = reading.readChoice(scanner);
            } catch (InputMismatchException e) {
                printUtils.printNotValidChoice();
                scanner.nextLine();
                continue;
            }

// лучше
    public void start() {

        while (true) {
            printUtils.printStartOrExitMenu();
            int answer = getChoice(scanner);
            ...
            
    private int getChoice(Scanner scanner) {
        while (!scanner.hasNextInt()) {
            scanner.nextLine();
            printUtils.printNotValidChoice();
        }
        return reading.readChoice(scanner);
    } 
// или же ловить исключение при парсинге инпута в самой утилите, и в случае чего возвращаться OptionalInt.empty()
```

- Логическая ошибка при валидации, сначала идет проверка на букву, а только потом на длину. Пользователь, который ввел "
  1a" получит не то сообщение об ошибке.

```
                if (!validation.isValidLetter(guessingLetter)) {
                    printUtils.printInputTypeError();
                    continue;
                }

                if (!validation.hasValidLength(input)) {
                    printUtils.printInputLengthError();
                    continue;
                }
```

- При получении значения для сложности используется тот же анти паттерн, решение такое же, что и выше.
- Почему-то `System.out.println("Good Bye!");` вызывается напрямую через `System`, хотя в проекте уже есть класс
  инкапсулирующий вывод в консоль. Решение - создать новый метод в `ReadingUtils` и использовать его.
- Использовать `ArrayList<Character>` для скрытого слова - _overhead_. Хватило бы `char[]` или `StringBuilder`.
- `Word`, `wrongLetters`, `difficulty`, `loseCount`, `winCount`, `lineToRead`, `hiddenWord` - все это представляет
  сущность `GameState`, а значит может быть вынесена в отдельный класс для удобства и инкапсуляции.

```java
public class GameState {
    private final String word;
    private final ArrayList<Character> hiddenWord;
    private final StringBuilder wrongLetters = new StringBuilder();
    private int loseCount;
    private int winCount;

    public GameState(String word, List<Character> hiddenWord) {
        this.word = word;
        this.hiddenWord = hiddenWord;
    }
    // конструктор, геттеры, сеттеры
}
```

- Семантически для коллекции с уникальными буквами подошел бы больше `Set<Character>`.

```
StringBuilder wrongLetters = new StringBuilder();
// лучше
Set<Character> wrongLetters = new LinkedHashSet<>();
```

- Имена переменных стоит делать более содержательными и понятными: `loseCount` -> `mistakeCount`, `winCount` ->
  `correctAnswersCount`, `guessingLetter` -> `guessLetter`
- Магическое число, стоит заменить на константу с содержательным именем:

```
 while (mistakeCount < 6)
 // лучше
 while (mistakeCount < MISTAKE_LIMIT)
```

- Стоит избегать написания комментариев, если вместо этого можно переписать код более понятным образом.
- Скрывай детали реализации, делай код проще, пользуйся вспомогательными переменными:

```java
public void start(){  // один большой божественный метод

        while(true){
            int answer;
            try {
                printUtils.printStartOrExitMenu();
                answer = reading.readChoice(scanner);
            } catch (InputMismatchException e) {
                printUtils.printNotValidChoice();
                scanner.nextLine();
                continue;
            }

            if(answer == QUIT){
                System.out.println("Good Bye!");
                break;
            }
            else if (answer != PLAY){
                printUtils.printNotValidChoice();
                continue;
            }
            //Game Starts
            StringBuilder wrongLetters = new StringBuilder();
            printUtils.printLetsStart();
            int difficulty;
            while(true) {
                try {
                    printUtils.printDifficultyMenu();
                    difficulty = reading.readChoice(scanner);
                    if(!validation.isValidChoice(difficulty)){
                        printUtils.printNotValidChoice();
                        continue;
                    }
                    break;
                } catch (InputMismatchException e) {
                    printUtils.printNotValidChoice();
                    scanner.nextLine();
                }
            }
            int loseCount = 0;
            int winCount = 0;
            int lineToRead = 0;
            String word = fileReading.readAWord(fileReading.getPathForDifficulty(difficulty)).toUpperCase();
            ArrayList<Character> hiddenWord = textProcessor.fillListWithUnderscore(word);
            fileReading.readGallowState(lineToRead);

            while (loseCount < 6) {
                if (winCount == word.length()) {
                    printUtils.printVictory(word);
                    break;
                }

                printUtils.printHiddenWord(hiddenWord);
                printUtils.printWrongLetters(wrongLetters);

                String input = reading.readInput(scanner);
                char guessingLetter = input.charAt(0);

                if (!validation.isValidLetter(guessingLetter)) {
                    printUtils.printInputTypeError();
                    continue;
                }

                if (!validation.hasValidLength(input)) {
                    printUtils.printInputLengthError();
                    continue;
                }

                if (!validation.isCorrectLetter(word, guessingLetter) && !validation.stringBuilderHasLetter(wrongLetters, guessingLetter)) {
                    lineToRead += 7;
                    loseCount++;
                    textProcessor.checkAndAppendWrongLetter(wrongLetters, guessingLetter);
                }

                if (!validation.listHasLetter(hiddenWord, guessingLetter)) {
                    winCount = textProcessor.putLettersToListAndCount(word, guessingLetter, hiddenWord, winCount);
                    fileReading.readGallowState(lineToRead);
                }
            }
            if (winCount != word.length()) {
                fileReading.readGallowState(lineToRead);
                printUtils.printGameOver(word);
            }
        }
    }
    
// лучше
public void start() { // стремимся разбить его на подметоды

        while (true) {
            printUtils.printStartOrExitMenu();
            int answer = getChoice(scanner);

            if (answer == PLAY) {
                startRound();
            } else if (answer == QUIT) {
                System.out.println("Good Bye!");
                break;
            } else {
              printUtils.printNotValidChoice();
            }
        }
    }

    private void startRound() {
        StringBuilder wrongLetters = new StringBuilder();
        printUtils.printLetsStart();
        int difficulty = getUserChoice();
        int mistakeCount = 0;
        int correctAnswersCount = 0;

        int lineToRead = 0;
        String word = fileReading.readAWord(fileReading.getPathForDifficulty(difficulty)).toUpperCase();
        ArrayList<Character> hiddenWord = textProcessor.fillListWithUnderscore(word);
        fileReading.readGallowState(lineToRead);

        startRoundLoop(mistakeCount, correctAnswersCount, word, hiddenWord, wrongLetters, lineToRead);
    }

    private void startRoundLoop(int mistakeCount, int correctAnswersCount, String word, ArrayList<Character> hiddenWord, StringBuilder wrongLetters, int lineToRead) {
        while (!isGameOver(mistakeCount)) {
            if (isWon(correctAnswersCount, word)) {
                printUtils.printVictory(word);
                break;
            }

            printUtils.printHiddenWord(hiddenWord);
            printUtils.printWrongLetters(wrongLetters);

            String input = reading.readInput(scanner);
            char guessLetter = getGuessLetter(input);

            if (!validation.isValidLetter(guessLetter)) { // в идеале еще и валидацию вынести в отдельный метод
                printUtils.printInputTypeError();
                continue;
            }

            if (!validation.hasValidLength(input)) {
                printUtils.printInputLengthError();
                continue;
            }

            if (isMistake(word, guessLetter, wrongLetters)) {
                lineToRead += 7;
                mistakeCount++;
                textProcessor.checkAndAppendWrongLetter(wrongLetters, guessLetter);
            }

            if (isCorrect(hiddenWord, guessLetter)) {
                correctAnswersCount = textProcessor.putLettersToListAndCount(word, guessLetter, hiddenWord, correctAnswersCount);
                fileReading.readGallowState(lineToRead);
            }
        }
        if (!isWon(correctAnswersCount, word)) {
            fileReading.readGallowState(lineToRead);
            printUtils.printGameOver(word);
        }
    }
```

- Стоит использовать коллекции через их интерфейсы (не только в этом месте, но и во всем проекте):

```java
// код для примера не из проекта для наглядности
ArrayList<Character> hiddenWord = new ArrayList();
// лучше
List<Character> hiddenWord = new ArrayList();
```

### FileReadingUtils

- Класс носит имя утилитарного, но сам не содержит свойственных ему статических методов. Решение: сделать все методы
  статическими, а также установить модификатор `final` вместе с `private` конструктором,
  свойственные признаки утилитарного класса, и использовать класс как утилитарный.
- `Random random = new Random();` - отсутствующий модификатор доступа.
- Методы `readAWord` и `readGallowStateLine` - каждый вызов заново вызывает `Files.readAllLines()`. Решение: стоит
  кэшировать.
- `int randomNumber = random.nextInt(20);` - захардкоженое значение - очень хрупко, если в файле будет не ровно 20 слов,
  то возможно исключение:

- Никогда не возвращай `null` - повышает вероятность _NPE_. Стоит либо возвращать Optional.empty(), либо пробрасывать
  исключение.

```java
    public String getPathForDifficulty(int levelOfDifficulty){

        switch(levelOfDifficulty){
            case 1 -> {
                return "resources/easy_words.txt";
            }
            case 2 -> {
                return "resources/medium_words.txt";
            }
            case 3 -> {
                return "resources/hard_words.txt";
            }
            default -> {
                System.out.println("Only specified numbers must be typed!");
                return null;
            }
        }

// лучше
    public String getPathForDifficulty(int levelOfDifficulty) {

        switch (levelOfDifficulty) {
            case 1 -> {
                return "resources/easy_words.txt";
            }
            case 2 -> {
                return "resources/medium_words.txt";
            }
            case 3 -> {
                return "resources/hard_words.txt";
            }
            default -> {
                throw new IllegalArgumentException("Only specified numbers must be typed!");
            }
        }  
```

- Можно вернуть без переменной - не игнорируй подсказки _idea_:

```java
            String word = Files.readAllLines(Paths.get(filePath)).get(randomNumber);
            return word;

// лучше
            return Files.readAllLines(Paths.get(filePath)).get(randomNumber);
```

- По какой-то причине метод `readGallowState` - создает свой же класс, для вызова метода:

```java
FileReadingUtils fileReading = new FileReadingUtils();
for (int i = lineToRead; i < lineToRead + 7; i++) {
    System.out.println(fileReading.readGallowStateLine(i));
}

// лучше
for (int i = lineToRead; i < lineToRead + 7; i++) {
    System.out.println(readGallowStateLine(i));
}
```

- `public String readGallowStateLine(int lineToRead)` - неправильный модификатор доступа, ведь метод используется только
  внутри класса.
- Метод `public void readGallowState(int lineToRead)` - нарушает _SRP_, ведь исходя из своего названия он должен что-то
  считать, а на самом деле еще и распечатывает в консоль. В целом, что-либо распечатывать в консоль надо через класс,
  который инкапсулирует это взаимодействие.
- `readGallowState` - сейчас метод в аргументах получает `int lineToRead`, что заставляет вызывающий слой знать какую
  именно строчку он хочет получить. Решением этой проблемы, например, было бы получение в аргументы порядкового номера
  виселицы, который соответствовал бы кол-ву ошибок, совершенных игроком. Это можно реализовать, например, с помощью
  `enum`. А сам метод можно будет переместить в тот же PrintUtils.

```java
    public void printGallowsStage(int lineToRead){
        FileReadingUtils fileReading = new FileReadingUtils();
        for (int i = lineToRead; i < lineToRead + 7; i++) {
            System.out.println(fileReading.readGallowStateLine(i));
        }
    }
    
// лучше
    public void printGallowsStage(int mistakes) {
        System.out.println(GallowsStages.fromOrdinal(mistakes + 1).getImage());
    }
    
    public enum GallowsStages {
    FIRST_STAGE("""
              +---+
              |   |
                  |
                  |
                  |
                  |
            =========
            """),
    ...


    private static final GallowsStages[] VALUES = values();
    private final String image;

    public static GallowsStages fromOrdinal(int ordinal) {
        if (ordinal < 0 || ordinal >= VALUES.length) {
            throw new IllegalArgumentException("Invalid ordinal: " + ordinal);
        }
        return VALUES[ordinal];
    }

    GallowsStages(String image) {
        this.image = image;
    }

    public String getImage() {
        return image;
    }
}

```

- Не стоит игнорировать исключения, надо пробросить дальше с сохранением `stacktrace`:

```java
        } catch (IOException e) {
            e.getMessage();
        }
       
// лучше 
        } catch (IOException e) {
            throw  new RuntimeException("Cannot read gallows state line",e);
        }
```

- По-английски правильнее будет `Gallows` - всегда в мж. числе `readGallowState` -> `readGallowsState`. Так же для
  метода, который должен вернуть какое-то значение более привычным было бы `get`. Поэтому `readGallowStateLine` ->
  `getGallowsStateLine`.

### PrintUtils

- Класс носит имя утилитарного, но сам не содержит свойственных ему статических методов. Решение: сделать все методы
  статическими, а также установить модификатор `final` вместе с `private` конструктором,
  свойственные признаки утилитарного класса, и использовать класс как утилитарный.

### ReadingUtils

- Класс носит имя утилитарного, но сам не содержит свойственных ему статических методов. Решение: сделать все методы
  статическими, а также установить модификатор `final` вместе с `private` конструктором,
  свойственные признаки утилитарного класса, и использовать класс как утилитарный.
- Не стоит игнорировать подсказки _idea_:

```java
        String input = scanner.next().toUpperCase();
        return input;
// лучше 
        return scanner.next().toUpperCase();
```

### TextProcessor

- Класс не хранит никакого состояния, а потому должен состоять из статических методов, иметь модификатор `final` и
  `private` конструктор.
- Неправильный модификатор доступа у `public ArrayList<Integer> letterIndexesInWord(String word, char letterToFind)`.
- `putLettersToListAndCount` - нарушает _SRP_, делает сразу две вещи. Как вариант решения: раскрывать буквы, а работу
  счетчиком оставить вызывающей стороне.
- Не игнорируй замечания _idea_:

```java
        for (int i = 0; i < indexes.size(); i++) {
            hiddenWord.set(indexes.get(i), guessingLetter);
            winCount++;
        }
// лучше
        for (Integer index : indexes) {
            hiddenWord.set(index, guessingLetter);
            winCount++;
        }
```

- Методу `checkAndAppendWrongLetter` больше подходит название `appendIfAbsent`.

### Validation

- Классы в _Java_ принято называть существительным соответственно `Validation` -> `Validator`.
- В классе смешана ответственность - есть валидация как инпута, так и логики игры. Решение - декомпозировать
  ответственность по классам.
- `listHasLetter`, `stringBuilderHasLetter` - названия привязаны к структурам данных, а не к смыслу. Лучше назвать в
  терминах игры:
  `isAlreadyGuessed`, `isAlreadyWrong`.

## Общее

- Стоит использовать _LTS_ версии _Java_ для проектов.
- Папку _.idea_ стоит также добавлять в _.gitignore_.
- Неплохо было бы оформлять коммиты в соответствии
  с [конвенцией](https://gist.github.com/qoomon/5dfcdf8eec66a051ecd85625518cfd13).
- Так же было бы полезно, на будущее, ознакомится с общепринятой Maven-структурой проекта.
- Стоит также, на будущее, ознакомится с системами сборки _Maven_/_Gradle_, иначе без _idea_ проект не соберется.
- Часто в проекте встречается несколько _Enter_'ов между методами - одного достаточно.
- Не стоит игнорировать замечания от *IDEA* - почти всегда это по делу, если ты специально не хочешь сделать по другому.
- Используй чаще форматирование (ctrl+alt+l в idea) - есть неаккуратные места.
- Магические числа/строки стоит заменять константами.

## Итог

- Считаю проект достоин рефакторинга, после которого можно смело идти дальше. Удачи!