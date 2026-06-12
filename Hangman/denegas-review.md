# Review на реализацию от [@denegas](https://github.com/denegas) проекта [Виселица](https://zhukovsd.github.io/java-backend-learning-course/projects/hangman/)

[Сама реализация](https://github.com/denegas/petProjects#)

## По реализации

Оцениваю исходя из описания ТЗ

Что улучшить:

- По ТЗ при старте приложение должно предложить начать новую игру или выйти. Сейчас сразу же начинается игра.
- Как вариант улучшения экспириенса пользователю - выводить использованные буквы.

Хорошо:

- на ввод хорошая валидация
- виселица рисуется
- сломать не получилось

## По коду

### class Main

- Важные для бизнес-логики строки лучше выносить в константу, избегая магических строк:

```
again = input.equalsIgnoreCase("y");

// лучше
private static final String ANSWER_TO_START_GAME = "y";

again = input.equalsIgnoreCase(ANSWER_TO_START_GAME)
```

- Ты ловишь конкретный `RuntimeException` от метода `scanner.nextLine()`, но не делаешь с ним чего-то специфичного,
  поэтому предлагаю вместо `NoSuchElementException` ловить `RuntimeException`. И для обработки исключений лучше
  соответствующие конструкции языка:

```
            try {
                String input = scanner.nextLine();
            } catch (NoSuchElementException err) {
                System.out.println("err with: " + err.getMessage());
            }
            
// лучше

            try {
                String input = scanner.nextLine();
            } catch (RuntimeException e) {
                throw new RuntimeException("Failed to read user answer", e); // или просто throw e
            }
```

- Объекты реализующие интерфейс Closeable или Autocloseable - могут быть использованы в конструкции try-with-resources,
  что удобнее для закрытия ресурсов, чем ручное `scanner.close()`.

```
Scanner scanner = new Scanner(System.in);
scanner.close();

// удобнее
try (Scanner scanner = new Scanner(System.in)) {
// ...
}
```

- В проекте уже есть класс для вывода в консоль, почему бы не добавить дополнительные методы для
  `System.out.println("Сыграть снова? (Y/N)");`, `System.out.println("Конец игры");`

```
System.out.println("Сыграть снова? (Y/N)");

// лучше
ConsoleOutput.printQuestion()
```

### class ConsoleOutput

- Вижу здесь более подходящим названием `ConsoleWriter`, `ConsoleRenderer`.
- Мне не очень нравится агрегация класса с word в виде зависимости, это заставляет методы, взаимодействующие с этим
  словом, быть нестатическими. Проще ведь будет просто передавать `String word` в аргументы метода.

```
public class ConsoleOutput {
    private final String word;

    public void HiddenWordLength(){
        System.out.println("Загаданное слово состоит из " + word.length() + " букв");
    }

// лучше, мне кажется зависимость излишней

    public static void HiddenWordLength(String word){
        System.out.println("Загаданное слово состоит из " + word.length() + " букв");
    }
```

- Если придерживаться моей предыдущей рекомендации, то можно сделать этот класс полностью утилитарным - сделать все
  методы статическими, спрятать конструктор, модификатор final в сигнатуре класса.

```
public class ConsoleOutput {
    private String word;
    public ConsoleOutput(String word){
        this.word = word;
    }
    
    public void print();
// лучше

public final class ConsoleOutput {
    private ConsoleOutput(){
    }
    
    public static void print...
    // etc
```  

- Метод `drawStickMan` немного обманывает - ведь при нуле ошибок он рисует одну виселицу, предлагаю переименовать в
  `drawGallows`.

```
public void drawStickMan(int triesCount){
        switch (triesCount){
            case 0: // рисует виселицу, а не стикмена
                System.out.println( 
                        "  +---+\n" +
                        "  |   |\n" +
                        "      |\n" +
                        "      |\n" +
                        "      |\n" +
                        "      |\n" +
                        "=========");
                break;
```

- `public void HiddenWordLength()` - методы стоит называть начиная со строчной буквы.

### class FileReader

- В java exception и error немного разные вещи, error связано с JVM - OutOfMemoryError, StackOverflowError etc. А
  IOException является исключения и лучше не путать эти понятия в коде. Также лучше использовать соответствующие
  конструкции языка вместо обычного sout.
- Также странно что метод все равно пытается вернуть список слов. IOException - недопустимая ситуация, программа не
  может продолжать работу, стоит выбросить исключение.

```
        } catch (IOException error) {
            System.out.println("Error with: " + error.getMessage());
            return arrayWords;
// лучше
        } catch (IOException e) {
            throw new RuntimeException("Failed to read from the file",e);
        }
```

- Слишком общее название метода `getArrayFromPath`, который дает список слов, почему бы не назвать - `getWordsFromFile`.
- Можно дополнительно спрятать конструктор и ограничить в наследовании от этого класса, чтобы соответствовать
  утилитарному классу.

### class GameLoop

- Для валидации строчек проще и надежнее использовать регулярные выражения, нежели перечислять весь русский алфавит.
- И еще имхо не очень корректно булевое выражение - смысл не совпадает с названием.
- Название метода `static boolean validateLetter` подразумевает, что он ничего не возвращает и просто выкидывает
  исключения при ошибки валидации, но это не так. Лучше назвать `isLetterNotValid`, а еще лучше возвращать не проверку
  на некорректность, а возврщать проверку на корректность и уже в коде с помощью оператора `!` делать отрицание, так
  будет читабельнее.

```
    private static final String RUSSIAN_ALPHABET = "абвгдеёжзийклмнопрстуфхцчшщъыьэюя";

    public static boolean validateLetter(String nowTry){
        return nowTry.length() != 1 || !RUSSIAN_ALPHABET.contains(nowTry);
    }
    // использование
            if (validateLetter(nowTry)) {
                consoleOutput.errorAlphabet();
                continue;
            }
                
// лучше
    private static final Pattern RUSSIAN_ALPHABET_PATTERN = Pattern.compile("^[а-яА-ЯёЁ]+$");

    public static boolean isValidLetter(String letter) {
        return letter.length() == 1 && RUSSIAN_ALPHABET_PATTERN.matcher(letter).matches();
    }
    
// использование становится нагляднее
            if (!isValidLetter(letter)) {
                consoleOutput.errorAlphabet();
                continue;
            }   
```

- Для удобства коллекции тоже можно вынести в константы.
- Для букв семантически вернее использовать тип `char`, а не `String`
- Зачем хранить найденные индексы, не проще ли верхнеуровнево хранить найденные буквы. Имхо возможно лишнее
  переусложнение.
- Метод `addFoundedIndexes` обманывает, т.к. он не просто добавляет новый индекс - он счала проверяет условие, и только
  если оно верно, то добавляет в список.

```
    private static void addFoundedIndexes(String hiddenWord,String nowTry,List<Integer> foundedIndexes){
        for (int i = 0; i < hiddenWord.length(); i++) { 
                if (String.valueOf(hiddenWord.charAt(i)).equals(nowTry)) { // добавление только с условием
                    foundedIndexes.add(i);
                }
            }
    }
```

- название локальной переменной, которая встречается везде `String nowTry` - проще назвать `letter`.
- Название `generateGuessedWord` обманывает, ведь он возвращает не загаданное слово, а слово-маску.

```
    private static String generateGuessedWord(String word,List<Integer> foundedIndexes){
        String foundedWord = "";
        for(int i =0;i<word.length();i++){
                if(foundedIndexes.contains(i)){
                    foundedWord += word.charAt(i);
                } else{
                    foundedWord += "*";
                }
            }
        return foundedWord; // возвращает слово с `*`, то есть маску, а не guessed word
    }
```

- В проекте встречаются разные названия - и Hidden word, и founded word, и путается Guessed Word, хочется единообразия.
- Магические числа стоит выносить в константы `while (tries < 6)`.

## Общее

- Используй только TLS версии java для разработки.
- Пользуйся [commit convention](https://habr.com/ru/articles/867012/) для именования коммитов.
- `.idea` папку добавляй в .gitignore
- Репу называй соответствуя содержимому, например - "Hangman".
- Не избегай подсказок idea, обычно они по делу.
- Используй форматирование idea - "ctrl + alt + L".

## Итог
- Для процедурного стиля неплохой проект, учти замечания и можешь идти дальше. Удачи! 