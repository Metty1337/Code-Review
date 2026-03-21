# Review в рамках учебной подписки на реализацию от [@MyNameIsN0body](https://github.com/MyNameIsN0body) проекта [Симуляция](https://zhukovsd.github.io/java-backend-learning-course/projects/simulation/)

[Сам проект](https://github.com/MyNameIsN0body/Simulation)

## Реализация

- Хорошо:
    - Красивое оформление меню.
    - Есть разные режимы игры.
    - Отображение состояния по ходу игры.
    - Есть возможность игроку самому размножить животных или добавить травы.
    - Нельзя ввести неправильно команду.
- Замечания:
    - Карта распечатывается не ровно. Не все спрайты одинаковой длины.
    - В ручном режиме нет концовки, после смерти всех существ все еще можно продолжать делать бессмысленные ходы.
    - Хищники и травоядные имеют один цвет спрайта из-за чего сливаются.

## По коду

### class Coordinates

- Класс может быть трансформирован в `record`, т.к. является иммутабельным.
- Использование обертки `Integer` для координат - излишество, ведь у нас не может быть координаты с `x = null`.
- На мой вкус - странный выбор пакета для класса (сейчас он в `entity`), мне кажется ему больше идет пакет `world`.
- В целом ничего лишнего - хорошо.

### class WorldMap

- Имена полей можно упростить:

```
worldLength -> length
worldWidth -> width
```

- Неправильное название - методы, которые начинаются с `set` в первую очередь сеттеры, а в данном случае он добавляет
  `Entity` на карту:

```
void setEntity(Coordinates coordinates, Entity entity)

// лучше
void putEntity(Coordinates coordinates, Entity entity)
```

- Почему бы сразу не передавать Coordinates в аргументах, тем самым делая метод проще:

```
    public boolean isCellEmpty(int x, int y) {
        return !entityMap.containsKey(new Coordinates(x, y));
    }
    
// лучше
    public boolean isCellEmpty(Coordinates coordinates) {
        return !entityMap.containsKey(coordinates);
    }
```

- Можно упростить метод, используй разные циклы по назначению:

```
    public Coordinates getRandomEmptyCoordinates() {
        for (int i = 0; i < 100; i++) {
            int x = random.nextInt(worldLength);
            int y = random.nextInt(worldWidth);
            if (isCellEmpty(x, y)) {
                return new Coordinates(x, y);
            }
        }
        return findAnyEmptyCoordinate();
    }

    private Coordinates findAnyEmptyCoordinate() {
        for (int y = 0; y < worldWidth; y++) {
            for (int x = 0; x < worldLength; x++) {
                if (isCellEmpty(x, y)) {
                    return new Coordinates(x, y);
                }
            }
        }
        throw new NoEmptyCellsException("Карта полностью заполнена");
    }

// лучше
    public Coordinates getRandomEmptyCoordinates() {
        int totalCells = worldLength * worldWidth;
        if (entityMap.size() >= totalCells) { // заранее проверяем не заполнена ли карта, чтобы лишний раз потом не проходить цикл
            throw new NoEmptyCellsException("Карта полностью заполнена");
        }

        while (true) {  // используем подходящий нам while вместо for
            int x = random.nextInt(worldLength);
            int y = random.nextInt(worldWidth);
            if (isCellEmpty(x, y)) {
                return new Coordinates(x, y);
            }
        }
    }
```

- Сообщение об исключении пиши только по-английски:

```
    private Coordinates findAnyEmptyCoordinate() {
// ...
        throw new NoEmptyCellsException("Карта полностью заполнена");
    }
    
        private Coordinates findAnyEmptyCoordinate() {
// ...
        throw new NoEmptyCellsException("Game map is completely full");
    }
```

- Нарушение Java-конвенции - приватный метод стоит выше публичного:

```
private Coordinates findAnyEmptyCoordinate() {}

public boolean isValidCoordinate(Coordinates coordinates) {}
```

- В классе есть метод `isValidCoordinate`, но он не используется в самом классе в подходящие для этого методы, где стоит
  проверить координату перед ее использованием:

```
    public void removeEntity(Coordinates coordinates) {
        entityMap.remove(coordinates);
    }
    
    // лучше
    public void removeEntity(Coordinates coordinates) {
        if (!isValidCoordinate(coordinates)){
            throw new IllegalArgumentException("Invalid coordinates");
        }
        entityMap.remove(coordinates);
    }
```

- Самый странный метод в классе `boolean moveEntity(Coordinates from, Coordinates to, Entity entity)` - нарушает SRP,
  т.к. карта не должна сама перемещать существ, для этого есть соответствующий Action по _ТЗ_. Также странная сигнатура
  метода - он возвращает `boolean`, хотя результат никогда не используется - стоит сделать метод void. А при
  невозможности хода выбрасывать исключение:

```
    public boolean moveEntity(Coordinates from, Coordinates to, Entity entity) {
        Optional <Entity> entityAtSource = getEntity(from);
        if (entityAtSource.isEmpty() || entityAtSource.get() != entity) {
            return false;
        }
        if (getEntity(to).isPresent() && !from.equals(to)) {
            return false;
        }

        removeEntity(from);
        setEntity(to, entity);
        return true;
    }
    
    // лучше
    public void moveEntity(Coordinates from, Coordinates to, Entity entity) {
        Optional <Entity> entityAtSource = getEntity(from);
        if (isНазваниеВсеОбъясняет(entity, entityAtSource)) { // скрываем детали, делаем код читабельнее
            throw new InvalidMoveException("сообщение об исключении по-английски");
        }
        if (isНазваниеВсеОбъясняет2(from, to)) { // скрываем детали, делаем код читабельнее
            throw new InvalidMoveException("сообщение об исключении по-английски");
        }
        removeEntity(from);
        setEntity(to, entity);
    }
```

- В методе `Optional<Coordinates> getEntityCoordinate(Entity entity)` не ясно почему используется `Optional`. Отсутствие
  координат у уже добавленной сущности в карту - недопустимая ситуация. Однозначно надо выбрасывать исключение. Причем
  во многих местах метод используется в виде `worldMap.getEntityCoordinate(creature).orElse(null)` - отчего пропадает
  всякий смысл от `Optional`.

### class BFSPathfinder:

- Если решил сделать класс утилитарным, то также следует добавить модификатор `final` классу. Но делать класс
  утилитарным спорное решение, ведь ты не сможешь выделить интерфейс для него и подменять в дальнейшем реализацию.
- Иметь свой связный список - хорошо, а также ему можно дать проще название и превратить в record:

```
    private static class PathNode {
        final Coordinates coordinates;
        final PathNode parent;

        PathNode(Coordinates coordinates, PathNode parent) {
            this.coordinates = coordinates;
            this.parent = parent;
        }
    }
    
    // лучше
    private record Node(Coordinates coordinates, Node parent) {
    }
```

- Не стоит передавать `null` в конструктор, лучше создать отдельный конструктор с одним аргументом:

```
        queue.add(new PathNode(start, null));

// лучше
        queue.add(new PathNode(start));
        
    private static class PathNode {
    // ...
        PathNode(Coordinates coordinates) {
            this.coordinates = coordinates;
            this.parent = null;
        }
    }
```

- В `List<Coordinates> findPath(WorldMap worldMap, Coordinates start, EntityType targetType)` используется `enum`
  `EntityType` для поиска экземпляра нужного класса, но нюанс в том, что реализовать это можно инструментами уже
  встроенными в язык, используя полиморфизм нашего абстрактного `Entity` и его наследников:

```
public static List<Coordinates> findPath(WorldMap worldMap, Coordinates start, EntityType targetType) {
// ...
                if (entity.isPresent() && entity.get().getType() == targetType) {
                    return reconstructPath(current);
                }
                
// лучше
    public static List<Coordinates> findPath(WorldMap worldMap, Coordinates start, Class<? extends Entity> targetClass) {
// ...
                if (entity.isPresent() && targetClass.isInstance(entity.get())) {
                    return reconstructPath(current);
                }
```
- Классы в _Java_ называют в стиле _UpperCamelCase_, это касается, в том числе сокращений.

### package entity

- В `enum EntityType` спрайты стоит сделать константами через конструктор, чтобы никто не мог их перезаписать, и назвать
  более конкретно - например`EntitySprite`:

```
public enum EntityType {
    HERBIVORE,
    PREDATOR,
// ...
    private final String sprite;

static {
    HERBIVORE.sprite = "🦌";//🐏"; //🐑";
// ...
}

// лучше
public enum EntitySprite {
    HERBIVORE(" "🦌";//🐏"; //🐑""),
// ...
    private final String sprite;

    EntitySprite(String sprite) {
        this.sprite = sprite;
    }

    public String getSprite() {
        return sprite;
    }
}
```

#### class Entity:

- Поле `protected final EntityType type;` используется в двух случаях:
    1. Для сравнения типов. Но в _Java_ можно передать типы в качестве параметров, необязательно иметь для этого
       отдельный `enum`.
    2. Для хранения спрайтов, что нарушает _SRP_. Стоит отделять состояние от представления, чтобы легко можно было
       подменить реализацию с консоли на другой любой.

- Нарушение _Java_-конвенции - конструктор идет ниже метода.

### package creatures

- В пакете накидано целых 15 классов из-за чего трудно разобраться. Лучше распределить по отдельным пакетам.

#### class Creature

- Нарушение _Java_-конвенции - конструктор идет ниже метода.
- 2 странных поля `protected int reproductionCooldown;
    protected int maxReproductionCooldown;` - семантически странно, что существо знает через какое время оно должно
  размножится. Я бы вынес это в другой класс.
- Если следовать _ТЗ_, то не хватает поля "Скорость".

#### class Herbivore

- Магические числа - стоит заменить на константу с содержательным именем:

```
    public Herbivore() {
        super(EntityType.HERBIVORE);
        this.energy = 9; 
        this.reproductionCooldown = 0;
        this.herbivoreMove = new HerbivoreMove();
        this.reproduction = new HerbivoreReproduction();
        this.herbivoreHunting = new HerbivoreHunting();
    }
    
// лучше    
    public Herbivore() {
        super(EntityType.HERBIVORE);
        this.energy = ИМЯ_ВСЕ_ОБЪСНЯЕТ;
        this.reproductionCooldown = ИМЯ_ВСЕ_ОБЪСНЯЕТ2;
        this.herbivoreMove = new HerbivoreMove();
        this.reproduction = new HerbivoreReproduction();
        this.herbivoreHunting = new HerbivoreHunting();
    }
```

- Присваивать значения полям все же лучше через конструктор класса выше, а не напрямую:

```
    public Herbivore() {
        super(EntityType.HERBIVORE);
        this.energy = 9;
        this.reproductionCooldown = 0;
        this.herbivoreMove = new HerbivoreMove();
        this.reproduction = new HerbivoreReproduction();
        this.herbivoreHunting = new HerbivoreHunting();
    }
// лучше
    public Herbivore() {
        super(EntityType.HERBIVORE, 9, 0);
        this.herbivoreMove = new HerbivoreMove();
        this.reproduction = new HerbivoreReproduction();
        this.herbivoreHunting = new HerbivoreHunting();
    }
```

- Также лучше не создавать свои зависимости внутри конструктора, это жесткая связность - невозможно подменить
  зависимости. Стоит обратиться к _Dependency Injection_ подходу, благодаря которому ясно видно какие зависимости есть у
  класса:

```
public Herbivore() {
    super(EntityType.HERBIVORE, 9, 0);
    this.herbivoreMove = new HerbivoreMove();
    this.reproduction = new HerbivoreReproduction();
    this.herbivoreHunting = new HerbivoreHunting();
}

// лучше
public Herbivore(HerbivoreMove herbivoreMove, HerbivoreReproduction herbivoreReproduction, HerbivoreHunting herbivoreHunting, int energy, int reproductionCooldown) {
    super(EntityType.HERBIVORE, energy, reproductionCooldown);
    this.herbivoreMove = herbivoreMove;
    this.reproduction = herbivoreReproduction;
    this.herbivoreHunting = herbivoreHunting;
}
```

#### class Predator

- По конструктору такой же комментарий, что и `Herbivore`.
- Старайся держать в одном методе один уровень абстракции, пользуйся вспомогательными методами, чтобы скрыть детали:

```
    @Override
    public void makeMove(WorldMap worldMap) {
        predatorMove.move(this, worldMap);
        MoveService.soulHarvester(this);
        if (this.getEnergy() <= 0) {
            worldMap.removeEntity(this);
        }
    }
    
// лучше
    @Override
    public void makeMove(WorldMap worldMap) {
        predatorMove.move(this, worldMap);
        MoveService.soulHarvester(this);
        if (isDead()) {
            die(worldMap);
        }
    }
```

- Нарушение _DRY_:

```
    @Override
    public void makeMove(WorldMap worldMap) {
// ...
        if (this.getEnergy() <= 0) {
            worldMap.removeEntity(this);
        }
    }
    
    @Override
    public void makeReproduce(WorldMap worldMap) {
// ...
            if (this.getEnergy() <= 0) {
                worldMap.removeEntity(this);
            }
        }
    }
```

#### interface Hunting

- Нетипичное название для интерфейса, обычно название описывает не процесс, а что объект умеет делать. Примеры из
  стандартной библиотеки: `Runnable`, `Comparable` etc. В данном случае лучше будет назвать `Hunter`.

#### interface Move

- То же самое для `Move`, хорошо подойдет `Movable`.

#### interface Reproduction

- То же самое и здесь, лучше подойдет `Reproducible`.

#### class BaseMove:

- Неверный модификатор доступа у `protected void moveToStep` - должен быть `private`.

#### class BaseHunting

- Стоит более точно давать имена аргументам:

```
public void hunt(Creature creature, WorldMap worldMap)

// лучше
public void hunt(Creature target, WorldMap worldMap)
``` 

- Неверный модификатор доступа у `protected void moveToTarget`- должен быть `private`.
- Если есть возможность избежать комментариев, переписав код лучше, то стоит этим воспользоваться:

```
    protected void moveToTarget(Creature creature, Coordinates targetStep, WorldMap worldMap) {
        Optional<Coordinates> currentPos = worldMap.getEntityCoordinate(creature);
        if (currentPos.isEmpty()) {
            return;
        }

        Optional<Entity> target = worldMap.getEntity(targetStep);

        if (target.isEmpty()) {
            // Просто двигаемся в пустую клетку
            worldMap.moveEntity(currentPos.get(), targetStep, creature);
        } else if (canEatTarget(target.orElse(null))) {
            // Съедаем цель
            onEatTarget(creature, targetStep, worldMap);
            creature.setEnergy(creature.getEnergy() + getEnergyGain());
        } else {
            // Цель не съедобна
            MoveService.moveRandomly(creature, worldMap);
        }
    }
    
// лучше
    protected void moveToTarget(Creature creature, Coordinates targetStep, WorldMap worldMap) {
        Optional<Coordinates> currentPosition = worldMap.getEntityCoordinate(creature);
        if (currentPosition.isEmpty()) {
            return;
        }

        Optional<Entity> target = worldMap.getEntity(targetStep);

        if (target.isEmpty()) {
            worldMap.moveEntity(currentPosition.get(), targetStep, creature);
        } else if (canEatTarget(target.get())) {
            eatTarget(creature, targetStep, worldMap);
        } else {
            MoveService.moveRandomly(creature, worldMap);
        }
    }
```

- Неправильная реакция на отсутствие координат, стоит выбрасывать исключение:

```
        Optional<Coordinates> currentPos = worldMap.getEntityCoordinate(creature);
        if (currentPos.isEmpty()) {
            return;
        }
```

#### class PredatorReproduction

- Метод `postReproductionActions` имеет слишком абстрактное название `applyReproductionCost` звучит лучше.
- Неясное имя поля `MIN_ENERGY` - лучше `ENERGY_COST_PER_REPRODUCTION`.

#### class MapConsoleRenderer

- Скрывай детали реализации, пользуйся вспомогательными методами:

```
    public void renderWorld(WorldMap worldMap) {
        int countSprite = worldMap.getWorldWidth();
        printBorderLine('╔', '╗', countSprite);
        for (int y = 0; y < countSprite; y++) {
            System.out.print(" ".repeat(INDENT_OUT) + "\u001B[36m║" + " ".repeat(INDENT_IN) + "\u001B[0m");
            for (int x = 0; x < countSprite; x++) {
                if (worldMap.isCellEmpty(x, y)) {
                    System.out.print(EntityType.EMPTY.getSprite());
                } else {
                    System.out.print(renderSprite(x, y, worldMap));
                }
            }
            System.out.println("\u001B[36m" + " ".repeat(INDENT_IN) + "║\u001B[0m");
        }
        printBorderLine('╚', '╝', countSprite);
    }
    
// лучше
    public void renderWorld(WorldMap worldMap) {
        int size = worldMap.getWorldWidth();
        printTopBorder(size);
        for (int y = 0; y < size; y++) {
            printLeftWall();
            printRowCells(y, size, worldMap);
            printRightWall();
        }
        printBottomBorder(size);
    }
 // вспомогательные методы
```

#### package services

- Все классы утилитарные, чистая процедурщина. Анти-паттерн в ООП - логика живет отдельно от данных. Решением вижу -
  разнести логику по владельцам данных или выделить отдельную сущность:
    - `DirectionService` - из него вполне получается хороший `enum`.
    - `FinderService` - может быть перенесен в тот же`EntityFinder`, где в композиции будет использоваться
      `BFSPathfinder` через интерфейс `PathFinder` - пример _ООП_.
    - `MoveService`:
        - Метод `soulHarvester` - должен относится к самим `Creature`.
        - `moveRandomly` - поведение существа, можно переместить к ним в `makeMove` или есть тот же `BaseMove`.
        - `canMove` - странный метод, содержит логику, которая должна пренадлежать самим существам (через тот же
          `canEatTarget`, который уже есть).
- `public static Coordinates findEmptyCellNear` - неверный модификатор доступа. А также никогда не возвращай `null`,
  т.к. это повышает шанс `NPE`.
- Неиспользуемый аргумент в `private static boolean creatureIsDead(Entity entity, WorldMap worldMap)`.
- Странный метод, который просто вызывает другой метод - проще переименовать оригинальный:
```
    public static boolean tryMove(Entity entity, Coordinates target, WorldMap worldMap) {
        return canMove(entity, target, worldMap);
    }
```
#### interface Actions:
- Лучше называть в единственном числе: `Action`.

#### class EatAction:
- Нарушение полиморфизма - у нас и так есть `Creature extends Entity`, мы можем просто пройтись по списку и поискать `Creature` и вызвать соответствующий метод. Именно за этим и нужен полиморфизм:
```
public class EatAction implements Actions{
    @Override
    public void execute(WorldMap worldMap) {
        List<Entity> allEntities = worldMap.getAllEntities();
        for (Entity entity : allEntities) {
            switch (entity.getType()) {
                case HERBIVORE -> {
                    ((Herbivore) entity).makeEat(worldMap);
                }
                case PREDATOR -> {
                    ((Predator) entity).makeEat(worldMap);
                }
            }
        }
    }
}

// лучше
  public class EatAction implements Actions {                                                                                                                                                             
      @Override                                                                                                                                                                                           
      public void execute(WorldMap worldMap) {                                                                                                                                                            
          for (Entity entity : worldMap.getAllEntities()) {                                                                                                                                               
              if (entity instanceof Creature creature) {                                                                                                                                                  
                  creature.makeEat(worldMap);                                                                                                                                                             
              }                                                                                                                                                                                           
          }                                                                                                                                                                                             
      }
  }

```

#### class InitAction:
- Конструктор стоит ниже метода - нарушение _Java_ конвенции.
- Приватный элемент стоит выше публичного - нарушение _Java_ конвенции.
- Два параллельных массива довольно хрупко - все держится на совпадении индексов: идеально для таких случаев подходит ассоциативный массив. Если хочется иметь свою фабрику, но в пределах одного класса, то в данном случае проще обойтись `Supplier<Entity>`. В целом, кмк, чище создать отдельный `EntityFactory`, но, так или иначе, можно улучшить текущий код:
```
public class InitAction implements Actions {
    private final int[] counts = new int[EntityType.values().length - 1];
    private final EntityFactory[] factories = new EntityFactory[EntityType.values().length - 1];

    private interface EntityFactory {
        Entity create();
    }

    public InitAction(int grassCount, int rockCount, int treeCount, int herbivoreCount, int predatorCount) {
        counts[EntityType.GRASS.ordinal()] = grassCount;
        counts[EntityType.ROCK.ordinal()] = rockCount;
        counts[EntityType.TREE.ordinal()] = treeCount;
        counts[EntityType.HERBIVORE.ordinal()] = herbivoreCount;
        counts[EntityType.PREDATOR.ordinal()] = predatorCount;

        factories[EntityType.GRASS.ordinal()] = Grass::new;
        factories[EntityType.ROCK.ordinal()] = Rock::new;
        factories[EntityType.TREE.ordinal()] = Tree::new;
        factories[EntityType.HERBIVORE.ordinal()] = Herbivore::new;
        factories[EntityType.PREDATOR.ordinal()] = Predator::new;
    }

// лучше    
 public class InitAction implements Actions {
      private final Map<Supplier<Entity>, Integer> entityCounts = new LinkedHashMap<>();

      public InitAction(int grassCount, int rockCount, int treeCount,
                        int herbivoreCount, int predatorCount) {
          entityCounts.put(Grass::new, grassCount);
          entityCounts.put(Rock::new, rockCount);
          entityCounts.put(Tree::new, treeCount);
          entityCounts.put(Herbivore::new, herbivoreCount);
          entityCounts.put(Predator::new, predatorCount);
      }
```

#### class MoveAction:
- То же самое, что и в EatAction - нарушение полиморфизма.

#### class ReproduceAction:
- То же самое - нарушение полиморфизма.

#### class GameMessenger:
- Метод `void showStatus` содержит логику симуляции. Метод должен принимать состояние игры и только рендерить, иначе нарушает `SRP`.
#### class Simulation:
- Поле без модификатора доступа - нарушение инкапсуляции:
```
    MapConsoleRenderer mapConsoleRenderer = new MapConsoleRenderer();
```
- `final` поля стоят ниже _instance_-полей - нарушение _Java_ конвенции:
```
    private final WorldMap worldMap;
    private int currentTurn = 0; <- ошибка

    MapConsoleRenderer mapConsoleRenderer = new MapConsoleRenderer(); <- должны стоять ниже

    private final List<Actions> initActions = new ArrayList<>();
    private final List<Actions> turnActions = new ArrayList<>();
```

- Лучше конечно не хардкодить, а передавать экшены и настройки в конструктор:
```
    public Simulation(int length, int width) {

        this.worldMap = new WorldMap(length, width);

        initActions.add(new InitAction(80, 20, 18, 20, 3));

        turnActions.add(new MoveAction());
        turnActions.add(new EatAction());
        turnActions.add(new GrassGrowthAction());
        turnActions.add(new ReproduceAction());
    }
```
- Неправильный модификатор доступа `public void initialize()`.
- Подразумевается, что initActions список и он будет расширен, так что лучше пройтись по всему списку, даже если там один элемент:
```
    public void initialize() {
        initActions.getLast().execute(worldMap);
    }
    
// лучше
    public void initialize() {
        for (Actions action : initActions) {
            action.execute(worldMap);
        }
    }
```
- Метод `public void startGame()` находится ниже многих приватных метод - нарушение _Java_ конвенции.
- Магические числа, стоит заменить внятной константой:
```
        if (mode.equals("1")) {
            stepByStepSimulation(scanner);
        } else if (mode.equals("2")) {
```
- Вместо анонимных классов лаконичнее использовать _method references_ или лямбды:
```
            Thread loopThread = new Thread(new Runnable() {
                @Override
                public void run() {
                    runSimulation();
                }
            });
// лучше
            Thread loopThread = new Thread(this::runSimulation);
```
- Метод `startSimulation` считаю должен быть удостоен отдельного класса. Иначе класс нарушает `SRP`.
- `volatile` поля для управления потоками сделаны статическими из-за чего запустить 2 симуляции не представляется возможным.
- Класс совмещает в себе 2 режима игры, предлагаю разделить эту ответственность между 2 классами.
### Общее
- Не стоит игнорировать замечания от *IDEA* - почти всегда это по делу, если ты специально не хочешь сделать по другому.
- Магические числа/строки стоит заменять константами.
- Используй чаще форматирование (ctrl+alt+l в idea) - есть неаккуратные места.
- Рекомендую изучить основные пункты _Java_ конвенцию.
 

## Итог
- Проект написан в процедурном стиле (Все сервисы - утилитарные классы, хранят логику, принадлежащую самим данным, которыми они оперируют, экшены реализованы через switch-case вместо полиморфизма, часто встречается нарушение инкапсуляции) в _ООП_ обертке (Встречаются _ООП_ элементы - наследование (Entity, Creature, Herbivore, Predator) и соответствующий полиморфизм, пускай и не всегда).
- Работающий проект, после соответствующего рефакторинга считаю можно идти дальше, удачи!