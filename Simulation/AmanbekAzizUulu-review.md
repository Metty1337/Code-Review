# Review на реализацию от [@AmanbekAzizUulu](https://github.com/AmanbekAzizUulu) проекта [Симуляция](https://zhukovsd.github.io/java-backend-learning-course/projects/simulation/)
[Сам проект](https://github.com/AmanbekAzizUulu/simulation-simple-implementation)

## Реализация
- Не раз замечал, что зайцы спавнятся сильно позже, не после смерти предыдущего поколеня, а уже после смерти волков, получается не очень сбалансировано.
- Не все эмоджи одной длины из-за чего в итоге получается не очень красивый рендер со съезжающими границами.
- Со старта симуляции, на сколько я понял, не гарантируется спаун животных из-за чего возникают десятки статических кадров.
- В целом симуляция рабочая, реализован бесконечный рендер новых ходов.

## По коду

### package model

#### package statics 

- Не стоит смешивать модель с представлением, это вызывает жесткую связность с рендером в консоль. В идеале модель вообще не знает каким образом ее будут рендерить, поскольку это ответственность рендера.
```java
public class Rock extends Terrain {

	public Rock () {
		super(EntityRepresentation.ROCK, false);
	}
}
```
#### package dynamics

- Класс Creture является родительским для Predator и Herbivore, но там находятся не общие черты этих двух подклассов (speed, hp), там находятся вообще все возможные поля. И получается ситуация, когда у нашего зайца появляется так называемый attackPower.
```java
public abstract class Creature extends Entity {
	private final Behavior behavior;
	private int speed;
	private int healthPoints;
	private int maxHealthPoints;
	private int attackPower;
	private int satiety;
	private int maxSatiety;
	private int nutritionValue;
```
- Инициализировать поля лучше через конструктор, а не через сеттер, если есть такая возможность:
```java
	public Herbivore (int speed) {
		super(EntityRepresentation.HERBIVORE, new HerbivoreBehavior());
		applyConfig(GameConfig.HERBIVORE_CONFIG);
		setSpeed(speed);
	}
	
	public Herbivore (int speed) {
		super(EntityRepresentation.HERBIVORE, new HerbivoreBehavior(), speed);
		applyConfig(GameConfig.HERBIVORE_CONFIG);
	}
```

- Также мне не очень нравится ситуация, когда мы через сеттеры применяем нужные настройки из конфига. Ни вижу причин ни делать это через конструктор.

- Не нашел метода makeMove() из ТЗ.

#### package map

- Для карты используется мапа `Map<Coordinate, Cell>`, где у `Cell` есть 2 поля и terrain и creature, что ни имеет смысла поскольку по ТЗ в клетке у нас может находится только один объект, поэтому предлагаю оставить просто Entity вместо Cell. 

Я догадался, что потребность в этом классе возникла из-за того, что мапа заполняется всегда полностью землей (и только по ней можем ходить) и вот эту логику можно упростить, полностью убрав этот класс (земли). В таком случае, например, если у нас в мапе нет нужного нам ключа (в пределах нашей карты) - это означает, что клетка свободна и по ней можно ходить, в тоже время для рендерера это означает, что клетку надо покрасить в коричневый. Это нам упощает логику и дизайн классов, а также показывает преимущество разделения состояния и представления. Пример - в классе `WorldMap` отпадет потребность в отдельных методах `setTerrain`, `placeCreature` - мы сможем полиморфно, как это и задумывалось, испольльзовать один `placeEntity`.

##### class WorldMap

- Принято, что приватные методы идут всегда после публичных.
- Мне не нравится, что карта заполняет сама себя - нарушение SRP. И по ТЗ отдельно прописана роль `initActions` по расставлению сущностей. Также магические числа, которые означают кол-во на спаун объектов лучше вынести в константы, а еще лучше эти значения принимать откуда-то из вне.
```java
	public WorldMap (int width, int height) {

		this.width = width;
		this.height = height;

		grid = new HashMap<>();

		initializeMap();
	}

	private void initializeMap () {
		fillWithSoil();
		buildLabyrinth();
	}

	private void fillWithSoil () {
		for (int y = 0; y < height; y++) {
			for (int x = 0; x < width; x++) {
				grid.put(new Coordinate(x, y), new Cell(new Soil()));
			}
		}
	}

	private void buildLabyrinth () {
		Random random = new Random();

		placeRandomTerrain(new Rock(), 10, random);
		placeRandomTerrain(new Tree(), 40, random);
		placeRandomTerrain(new Water(), 50, random);
	}
```

- Используется Optional, чтобы дальше пробросить null, null никогда нельзя возвращать:
```java
	public Creature getCreature (Coordinate coordinate) {
		return Optional.ofNullable(grid.get(coordinate))
		               .map(Cell::getCreature)
		               .orElse(null);
	}
```

- Никогда нельзя возвращать null:
```java
	public Coordinate getCreatureCoordinates (Creature creature) {
		for (var entry : grid.entrySet()) {
			if (entry.getValue()
			         .getCreature() == creature) {
				return entry.getKey();
			}
		}
		return null;
	}
```

- Если у нас по координате никто не находится на карте, то мы должны выкинуть исключение, а не возвращать false:
```java
	public boolean isPassable (Coordinate coordinate) {
		if (!isInsideTheWorldMap(coordinate)) {
			return false;
		}

		Cell cell = getCell(coordinate);
		if (cell == null) {
			return false;
		}

		Terrain terrain = cell.getTerrain();
		if (terrain == null) {
			return false;
		}
		return terrain.isPassable();
	}
```

#### package map
- Класс PathFinder нигде используется.

#### package target

- Хорошая идея передавать в аргументы предикат и тем самым реализуя логику хода разных существ у них в классе.

### package renderer

- Главный метод получился слишком раздутым, хотелось бы его как-то декомпозировать.

### package logging

- Логирование через ивенты мне кажется каким-то супер оверхедом. Как будто проще через SLF4J.

### package action

- В моем понимании удачный пакет Action состоит из SpawnAction - спавним на карте объекты/существа на усмотрение, MoveAction - мы у каждого существа вызываем метод makeMove() (по ТЗ) - если вблизи таргет, то съедаем, если нет, то идем за ним. И на этом все. В этом же проекте я наблюдаю еще помимо Action - еще пакет behavior и целый огромный пакет interactions, который наполнен некими commands. И эти абстракции ради абстракции, паттерны ради паттернов нового функционала тоже не приносят, лишь сильно усложняют код.


### package simulation

#### class SimulationEngine

- Класс SimulationEngine - класс Simulation из ТЗ, в ТЗ этот класс описан лучше всего.
- Отсутствует счетчик ходов по ТЗ.
- Исключения лучше не игнорировать, и писать о нем на английском:
```java
		} catch (IOException e) {
			System.err.println("Не удалось настроить логирование в файл: " + e.getMessage());
		}
```
- Отсутствует метод или его аналог pauseSimulation по ТЗ.
- При InterruptedException обязательно следует восстанавливать флаг:
```java
		} catch (InterruptedException e) {
			e.printStackTrace();
			
// лучше
		} catch (InterruptedException e) {
			Thread.currentThread().interrupt();
```
- По ТЗ в этом классе должно быть 2 списка Actions - initActions, которые вызывались бы перед стартом симуляции и turnActions, которые исполнялись бы каждый ход. Ни того ни другого нет. А для исполнения Action они создаются на месте:
```java
	private void performOneStep () {
		new HungerAction().perform(worldMap);
		moveCreaturesAction.perform(worldMap);
		attackCreatureAction.perform(worldMap);
		new SpawnEntitiesAction().perform(worldMap);
	}
	
// как должно быть по ТЗ

    private void performOneStep() {
        for (Action action : turnActions) {
            action.perform(worldMap);
        }
    }
```

- В конструктор не передается ничего, каждый зависимость зашита внутрь, никак нельзя изменить, лучше конечно передавать из вне хотя бы параметры карты:
```java
	public SimulationEngine () {
		InteractionSystem interactionSystem = new InteractionSystem();

		this.worldMap = new WorldMap(15, 15);

		this.moveCreaturesAction = new MoveCreaturesAction(interactionSystem);
		this.attackCreatureAction = new AttackCreatureAction(interactionSystem);

		this.renderer = new ConsoleRenderer();

		EventBus.getInstance().register(new LoggingEventListener());
	}
	
	public SimulationEngine (int width, int height) {
		InteractionSystem interactionSystem = new InteractionSystem();

		this.worldMap = new WorldMap(width, height);

		this.moveCreaturesAction = new MoveCreaturesAction(interactionSystem);
		this.attackCreatureAction = new AttackCreatureAction(interactionSystem);

		this.renderer = new ConsoleRenderer();

		EventBus.getInstance().register(new LoggingEventListener());
	}
```
- Также не рекомендуется импортировать статические методы, пусть лучше будет сразу виден источник метода:
```java
sleep(100);
// лучше
Thread.sleep(100);
```

## Общее
- Не игнорируй подсказки idea.
- Никогда не закрывай ТЗ.
- Не усложняй себе задачу.
- Советую обратить внимание на эталонную реализацию.

## Итог

В целом это ООП, хотя были нарушения полиморфизма. Главное это читать ТЗ и не оверинженерить. Учти нюансы и можешь смело идти дальше. Удачи!