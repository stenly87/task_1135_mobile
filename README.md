В этом руководстве мы создадим приложение для отслеживания баланса игроков в настольной игре Монополия.
Приложение будет состоять из одного Activity с двумя фрагментами.

---

## Шаг 0: Создание нового проекта

1. Откройте **Android Studio**
2. Нажмите **"New Project"**
3. Выберите шаблон **"Empty Views Activity"**
4. Настройте проект:
   - **Name**: MonopolyBalanceTracker
   - **Package name**: com.example.monopolybalance
   - **Language**: Kotlin
   - **Minimum SDK**: API 24 (Android 7.0) или выше
5. Нажмите **"Finish"**

---

## Шаг 1: Добавление зависимостей

Откройте файл `build.gradle.kts` и добавьте следующие зависимости в секцию `plugins`:

```gradle
alias(libs.plugins.kotlin.parcelize)
```

Откройте файл `libs.versions.toml` и добавьте следующую строку в секцию `plugins`:
```gradle
kotlin-parcelize = { id = "org.jetbrains.kotlin.plugin.parcelize", version.ref = "kotlin" }
```
и следующую строку  в секцию `versions`
```gradle
kotlin = "2.4.20"
```

Синхронизируйте проект, нажав **"Sync Now"**.

---

## Шаг 2: Создание модели данных

Создайте новый файл `Player.kt` в папке `app/src/main/java/com/example/monopolybalance`:

```kotlin
package com.example.monopolybalance

@Parcelize
data class Player(
    val id: Int,
    var name: String,
    var balance: Int = 1500
): Parcelable
```
Parcelize включает возможность сериализации объекта
---

## Шаг 3: Создание фрагментов

Создайте новые фрагменты (Fragment vith ViewModel): SetupFragment и GameFragment

---

## Шаг 4: Измените ViewModel для первого экрана

Откройте файл `SetupViewModel.kt`:

### Метод 1: Инициализация списка имен игроков

Этот метод создает изменяемый список для хранения имен добавленных игроков:

```kotlin
private val _playerNames = MutableLiveData<MutableList<String>>(mutableListOf())
val playerNames: LiveData<MutableList<String>> = _playerNames
```

### Метод 2: Свойство для отслеживания количества игроков

Это вычисляемое свойство возвращает текущее количество добавленных игроков:

```kotlin
val playerCount: LiveData<Int> = _playerNames.map { it.size }
```

### Метод 3: Функция проверки возможности добавления игрока

Данная функция проверяет, можно ли добавить нового игрока (максимум 4):

```kotlin
fun canAddPlayer(): Boolean {
    return _playerNames.value?.size ?: 0 < 4
}
```

### Метод 4: Функция добавления игрока

Эта функция добавляет имя игрока в список, если это возможно:

```kotlin
fun addPlayer(name: String): Boolean {
    if (!canAddPlayer() || name.isBlank()) {
        return false
    }
    
    val currentList = _playerNames.value ?: mutableListOf()
    currentList.add(name)
    _playerNames.value = currentList
    
    return true
}
```

### Метод 5: Функция проверки возможности начала игры

Функция проверяет, добавлено ли минимум 2 игрока для начала игры:

```kotlin
fun canStartGame(): Boolean {
    return (_playerNames.value?.size ?: 0) >= 2
}
```

### Метод 6: Функция получения всех игроков для передачи во второй экран

Этот метод преобразует список имен в список объектов Player:

```kotlin
fun getPlayersForGame(): List<Player> {
    return _playerNames.value?.mapIndexed { index, name ->
        Player(id = index + 1, name = name, balance = 1500)
    } ?: emptyList()
}
```

---

## Шаг 5: Изменение ViewModel для второго экрана

Откройте файл `GameViewModel.kt`:

### Метод 1: Хранилище списка игроков

Это свойство хранит список всех игроков игры:

```kotlin
private val _players = MutableLiveData<List<Player>>()
val players: LiveData<List<Player>> = _players
```

### Метод 2: Текущий выбранный игрок

Это свойство отслеживает ID выбранного в спиннере игрока:

```kotlin
private val _selectedPlayerId = MutableLiveData<Int>(1)
val selectedPlayerId: LiveData<Int> = _selectedPlayerId
```

### Метод 3: Получение текущего игрока

Функция возвращает объект игрока по его ID:

```kotlin
fun getCurrentPlayer(): Player? {
    return _players.value?.find { it.id == _selectedPlayerId.value }
}
```

### Метод 4: Обновление выбранного игрока

Эта функция обновляет ID выбранного игрока при изменении в спиннере:

```kotlin
fun setSelectedPlayer(playerId: Int) {
    _selectedPlayerId.value = playerId
}
```

### Метод 5: Инициализация игры со списком игроков

Данный метод устанавливает начальный список игроков при переходе на экран игры:

```kotlin
fun initializeGame(players: List<Player>) {
    _players.value = players
}
```

### Метод 6: Добавление денег игроку

Функция увеличивает баланс выбранного игрока на указанную сумму:

```kotlin
fun addMoney(amount: Int) {
    val currentPlayer = getCurrentPlayer() ?: return
    
    val updatedPlayers = _players.value?.map { player ->
        if (player.id == currentPlayer.id) {
            player.copy(balance = player.balance + amount)
        } else {
            player
        }
    }
    
    _players.value = updatedPlayers !!
}
```

### Метод 7: Перевод денег между игроками или в банк

Эта функция снимает деньги с одного игрока и добавляет другому (или в банк):

```kotlin
fun transferMoney(fromPlayerId: Int, toPlayerId: Int?, amount: Int): Boolean {
    val fromPlayer = _players.value?.find { it.id == fromPlayerId } ?: return false
    
    // Проверка достаточности средств
    if (fromPlayer.balance < amount) {
        return false
    }
    
    val updatedPlayers = _players.value?.map { player ->
        when {
            player.id == fromPlayerId -> {
                player.copy(balance = player.balance - amount)
            }
            toPlayerId != null && player.id == toPlayerId -> {
                player.copy(balance = player.balance + amount)
            }
            else -> player
        }
    }
    
    _players.value = updatedPlayers !!
    return true
}
```

### Метод 8: Получение списка имен для спиннера

Функция возвращает список имен всех игроков для отображения в выпадающем списке:

```kotlin
fun getPlayerNames(): List<String> {
    return _players.value?.map { it.name } ?: emptyList()
}
```

### Метод 9: Получение списка получателей для перевода

Этот метод создает список для спиннера получателей, включая опцию "Банк":

```kotlin
fun getRecipientsList(): List<String> {
    val recipients = mutableListOf("Банк")
    recipients.addAll(getPlayerNames())
    return recipients
}
```

---

## Шаг 6: Создание навигационного графа

Создайте файл `nav_graph.xml` в папке `res/navigation`:

### Структура навигации

Навигационный граф определяет два фрагмента и переход между ними:

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/nav_graph"
    app:startDestination="@id/setupFragment">

    <fragment
        android:id="@+id/setupFragment"
        android:name="com.example.monopolybalance.SetupFragment"
        android:label="SetupFragment">
        <action
            android:id="@+id/action_setup_to_game"
            app:destination="@id/gameFragment" />
    </fragment>

    <fragment
        android:id="@+id/gameFragment"
        android:name="com.example.monopolybalance.GameFragment"
        android:label="GameFragment" />

</navigation>
```

---

## Шаг 7: Изменение MainActivity

Откройте `activity_main.xml`, уберите текстовое поле, скачайте в палитре элемент NavHostFragment и расположите его в корне:

Должен получиться следующий код:

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/nav_host_fragment"
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:defaultNavHost="true"
        app:navGraph="@navigation/nav_graph" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

---

## Шаг 8: Изменение первого фрагмента - SetupFragment

Откройте файл `fragment_setup.xml` в папке `res/layout`:

### Layout фрагмента настройки

Разметка включает поле ввода имени, список добавленных игроков и кнопки управления:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp"
    android:gravity="center_horizontal">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Создание игроков"
        android:textSize="24sp"
        android:textStyle="bold"
        android:layout_marginBottom="24dp" />

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Добавьте от 2 до 4 игроков"
        android:textSize="16sp"
        android:layout_marginBottom="16dp" />

    <ListView
        android:id="@+id/lvPlayers"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:layout_marginBottom="16dp" />

    <EditText
        android:id="@+id/etPlayerName"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Имя игрока"
        android:inputType="textPersonName"
        android:maxLength="20"
        android:layout_marginBottom="8dp" />

    <Button
        android:id="@+id/btnAddPlayer"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Добавить игрока" />

    <TextView
        android:id="@+id/tvPlayerCount"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Добавлено игроков: 0"
        android:layout_marginTop="8dp" />

    <Button
        android:id="@+id/btnStartGame"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Начать игру"
        android:enabled="false"
        android:layout_marginTop="16dp" />

</LinearLayout>
```

### Изменение файла SetupFragment.kt

#### Метод 1: Объявление переменных для UI элементов

Здесь объявляются все необходимые view-элементы фрагмента:

```kotlin
private lateinit var etPlayerName: EditText
private lateinit var btnAddPlayer: Button
private lateinit var btnStartGame: Button
private lateinit var lvPlayers: ListView
private lateinit var tvPlayerCount: TextView
```

#### Метод 2: Инициализация ViewModel

Этот метод создает экземпляр ViewModel, привязанный к жизненному циклу активности:

```kotlin
private val viewModel: SetupViewModel by activityViewModels()
```

#### Метод 3: Адаптер для списка игроков

Создается адаптер для отображения списка добавленных игроков:

```kotlin
private val adapter by lazy {
    ArrayAdapter(requireContext(), android.R.layout.simple_list_item_1, mutableListOf<String>())
}
```

#### Метод 4: Функция инициализации view-элементов

Данная функция связывает переменные с элементами разметки:

```kotlin
private fun initializeViews(view: View) {
    etPlayerName = view.findViewById(R.id.etPlayerName)
    btnAddPlayer = view.findViewById(R.id.btnAddPlayer)
    btnStartGame = view.findViewById(R.id.btnStartGame)
    lvPlayers = view.findViewById(R.id.lvPlayers)
    tvPlayerCount = view.findViewById(R.id.tvPlayerCount)
    
    lvPlayers.adapter = adapter
}
```

#### Метод 5: Настройка наблюдателей LiveData

Эти наблюдатели реагируют на изменения данных в ViewModel:

```kotlin
private fun setupObservers() {
    viewModel.playerNames.observe(viewLifecycleOwner) { names ->
        adapter.clear()
        adapter.addAll(names)
        adapter.notifyDataSetChanged()
    }
    
    viewModel.playerCount.observe(viewLifecycleOwner) { count ->
        tvPlayerCount.text = "Добавлено игроков: $count"
        btnStartGame.isEnabled = viewModel.canStartGame()
    }
}
```

#### Метод 6: Обработчик добавления игрока

Эта функция обрабатывает нажатие кнопки добавления игрока:

```kotlin
private fun setupAddPlayerListener() {
    btnAddPlayer.setOnClickListener {
        val name = etPlayerName.text.toString().trim()
        
        if (viewModel.addPlayer(name)) {
            etPlayerName.text.clear()
            Toast.makeText(context, "Игрок добавлен", Toast.LENGTH_SHORT).show()
        } else {
            Toast.makeText(context, "Невозможно добавить игрока", Toast.LENGTH_SHORT).show()
        }
    }
}
```

#### Метод 7: Обработчик начала игры

Функция осуществляет навигацию ко второму фрагменту с передачей данных:

```kotlin
private fun setupStartGameListener() {
    btnStartGame.setOnClickListener {
        val players = viewModel.getPlayersForGame()
        
        val bundle = Bundle().apply {
            putParcelableArrayList("players", ArrayList(players))
        }
        
        findNavController().navigate(
            R.id.action_setup_to_game,
            bundle
        )
    }
}
```

Замените содержимое метода onCreateView вызовом созданных методов
```kotlin
	val view = inflater.inflate(R.layout.fragment_game, container, false)
	initializeViews(view)
	initializeGame()
	setupSpinnerListeners()
	setupObservers()
	setupButtonListeners()

	return view
```

---

## Шаг 9: Изменение второго фрагмента - GameFragment

Откройте файл `fragment_game.xml` в папке `res/layout`:

### Layout фрагмента игры

Разметка включает спиннеры для выбора игрока и получателя, кнопки +/-, формы для ввода сумм и таблицу балансов:

```xml
<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="16dp">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Игра"
            android:textSize="24sp"
            android:textStyle="bold"
            android:layout_gravity="center_horizontal"
            android:layout_marginBottom="16dp" />

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Выберите игрока:"
            android:textSize="16sp"
            android:layout_marginBottom="8dp" />

        <Spinner
            android:id="@+id/spinnerPlayers"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp" />

        <TextView
            android:id="@+id/tvCurrentBalance"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Баланс: 1500 золотых"
            android:textSize="18sp"
            android:textStyle="bold"
            android:layout_gravity="center_horizontal"
            android:layout_marginBottom="24dp" />

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:gravity="center">

            <Button
                android:id="@+id/btnAddMoney"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:text="+"
                android:textSize="24sp"
                android:layout_marginEnd="8dp" />

            <Button
                android:id="@+id/btnRemoveMoney"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:text="-"
                android:textSize="24sp"
                android:layout_marginStart="8dp" />

        </LinearLayout>

        <LinearLayout
            android:id="@+id/layoutAddMoney"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:visibility="gone"
            android:layout_marginTop="16dp"
            android:padding="16dp"
            android:background="#E8F5E9">

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Добавить деньги"
                android:textSize="18sp"
                android:textStyle="bold"
                android:layout_marginBottom="8dp" />

            <EditText
                android:id="@+id/etAddAmount"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="Сумма"
                android:inputType="number"
                android:layout_marginBottom="8dp" />

            <Button
                android:id="@+id/btnSaveAdd"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Сохранить" />

        </LinearLayout>

        <LinearLayout
            android:id="@+id/layoutRemoveMoney"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:visibility="gone"
            android:layout_marginTop="16dp"
            android:padding="16dp"
            android:background="#FFEBEE">

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Перевести деньги"
                android:textSize="18sp"
                android:textStyle="bold"
                android:layout_marginBottom="8dp" />

            <EditText
                android:id="@+id/etRemoveAmount"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="Сумма"
                android:inputType="number"
                android:layout_marginBottom="8dp" />

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Кому перевести:"
                android:layout_marginBottom="4dp" />

            <Spinner
                android:id="@+id/spinnerRecipient"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginBottom="8dp" />

            <Button
                android:id="@+id/btnSaveTransfer"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Сохранить" />

        </LinearLayout>

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Балансы игроков:"
            android:textSize="18sp"
            android:textStyle="bold"
            android:layout_marginTop="24dp"
            android:layout_marginBottom="8dp" />

        <ListView
            android:id="@+id/lvAllBalances"
            android:layout_width="match_parent"
            android:layout_height="200dp" />

    </LinearLayout>

</ScrollView>
```

### Изменение файла GameFragment.kt

#### Метод 1: Объявление UI переменных

Объявляются все элементы интерфейса второго фрагмента:

```kotlin
private lateinit var spinnerPlayers: Spinner
private lateinit var spinnerRecipient: Spinner
private lateinit var tvCurrentBalance: TextView
private lateinit var btnAddMoney: Button
private lateinit var btnRemoveMoney: Button
private lateinit var layoutAddMoney: LinearLayout
private lateinit var layoutRemoveMoney: LinearLayout
private lateinit var etAddAmount: EditText
private lateinit var btnSaveAdd: Button
private lateinit var etRemoveAmount: EditText
private lateinit var btnSaveTransfer: Button
private lateinit var lvAllBalances: ListView
```

#### Метод 2: Переменные для отслеживания выбранных значений

Эти переменные хранят ID выбранного игрока и получателя:

```kotlin
private var selectedPlayerId: Int = 1
private var selectedRecipientIndex: Int = 0
```

#### Метод 3: Инициализация ViewModel

Создается экземпляр GameViewModel:

```kotlin
private val viewModel: GameViewModel by activityViewModels()
```

#### Метод 4: Адаптеры для спиннеров и списка

Создаются адаптеры для отображения данных в спиннерах и списке балансов:

```kotlin
private val playersAdapter by lazy {
    ArrayAdapter(requireContext(), android.R.layout.simple_spinner_item, mutableListOf<String>())
}

private val recipientsAdapter by lazy {
    ArrayAdapter(requireContext(), android.R.layout.simple_spinner_item, mutableListOf<String>())
}

private val balancesAdapter by lazy {
    ArrayAdapter(requireContext(), android.R.layout.simple_list_item_2, 
        android.R.id.text1, mutableListOf<String>())
}
```

#### Метод 5: Функция инициализации view-элементов

Связывает переменные с элементами разметки:

```kotlin
private fun initializeViews(view: View) {
    spinnerPlayers = view.findViewById(R.id.spinnerPlayers)
    spinnerRecipient = view.findViewById(R.id.spinnerRecipient)
    tvCurrentBalance = view.findViewById(R.id.tvCurrentBalance)
    btnAddMoney = view.findViewById(R.id.btnAddMoney)
    btnRemoveMoney = view.findViewById(R.id.btnRemoveMoney)
    layoutAddMoney = view.findViewById(R.id.layoutAddMoney)
    layoutRemoveMoney = view.findViewById(R.id.layoutRemoveMoney)
    etAddAmount = view.findViewById(R.id.etAddAmount)
    btnSaveAdd = view.findViewById(R.id.btnSaveAdd)
    etRemoveAmount = view.findViewById(R.id.etRemoveAmount)
    btnSaveTransfer = view.findViewById(R.id.btnSaveTransfer)
    lvAllBalances = view.findViewById(R.id.lvAllBalances)
    
    spinnerPlayers.adapter = playersAdapter
    spinnerRecipient.adapter = recipientsAdapter
    lvAllBalances.adapter = balancesAdapter
}
```

#### Метод 6: Получение переданных данных

Эта функция извлекает список игроков из Bundle:

```kotlin
private fun getArgumentsPlayers(): List<Player>? {
    return arguments?.getParcelableArrayList<Player>("players")
}
```

#### Метод 7: Инициализация игры с переданными данными

Функция передает список игроков в ViewModel и заполняет спиннеры:

```kotlin
private fun initializeGame() {
    val players = getArgumentsPlayers()
    if (players != null) {
        viewModel.initializeGame(players)
        updateSpinners()
    }
}
```

#### Метод 8: Обновление содержимого спиннеров

Эта функция обновляет данные в спиннерах игроков и получателей:

```kotlin
private fun updateSpinners() {
    playersAdapter.clear()
    playersAdapter.addAll(viewModel.getPlayerNames())
    playersAdapter.notifyDataSetChanged()
    
    recipientsAdapter.clear()
    recipientsAdapter.addAll(viewModel.getRecipientsList())
    recipientsAdapter.notifyDataSetChanged()
}
```

#### Метод 9: Настройка слушателей спиннеров

Настраиваются обработчики выбора элементов в спиннерах:

```kotlin
private fun setupSpinnerListeners() {
    spinnerPlayers.onItemSelectedListener = object : AdapterView.OnItemSelectedListener {
        override fun onItemSelected(parent: AdapterView<*>, view: View?, position: Int, id: Long) {
            selectedPlayerId = position + 1
            viewModel.setSelectedPlayer(selectedPlayerId)
            updateCurrentBalance()
        }
        
        override fun onNothingSelected(parent: AdapterView<*>) {}
    }
    
    spinnerRecipient.onItemSelectedListener = object : AdapterView.OnItemSelectedListener {
        override fun onItemSelected(parent: AdapterView<*>, view: View?, position: Int, id: Long) {
            selectedRecipientIndex = position
        }
        
        override fun onNothingSelected(parent: AdapterView<*>) {}
    }
}
```

#### Метод 10: Настройка наблюдателей LiveData

Наблюдатели отслеживают изменения списка игроков:

```kotlin
private fun setupObservers() {
    viewModel.players.observe(viewLifecycleOwner) { players ->
        updateBalancesList()
        updateCurrentBalance()
    }
}
```

#### Метод 11: Обновление отображения текущего баланса

Функция обновляет текст с балансом выбранного игрока:

```kotlin
private fun updateCurrentBalance() {
    val player = viewModel.getCurrentPlayer()
    if (player != null) {
        tvCurrentBalance.text = "Баланс: ${player.balance} золотых"
    }
}
```

#### Метод 12: Обновление списка всех балансов

Эта функция обновляет список балансов всех игроков:

```kotlin
private fun updateBalancesList() {
    val players = viewModel.players.value ?: return
    
    val balances = players.map { "${it.name}: ${it.balance} золотых" }
    
    balancesAdapter.clear()
    balancesAdapter.addAll(balances)
    balancesAdapter.notifyDataSetChanged()
}
```

#### Метод 13: Показать форму добавления денег

Функция отображает форму для добавления денег и скрывает форму перевода:

```kotlin
private fun showAddMoneyForm() {
    layoutAddMoney.visibility = View.VISIBLE
    layoutRemoveMoney.visibility = View.GONE
    etAddAmount.requestFocus()
}
```

#### Метод 14: Показать форму перевода денег

Функция отображает форму для перевода денег и скрывает форму добавления:

```kotlin
private fun showRemoveMoneyForm() {
    layoutRemoveMoney.visibility = View.VISIBLE
    layoutAddMoney.visibility = View.GONE
    etRemoveAmount.requestFocus()
}
```

#### Метод 15: Сохранить добавление денег

Эта функция обрабатывает добавление денег выбранному игроку:

```kotlin
private fun saveAddMoney() {
    val amountStr = etAddAmount.text.toString()
    
    if (amountStr.isEmpty()) {
        Toast.makeText(context, "Введите сумму", Toast.LENGTH_SHORT).show()
        return
    }
    
    val amount = amountStr.toIntOrNull()
    if (amount == null || amount <= 0) {
        Toast.makeText(context, "Введите корректную сумму", Toast.LENGTH_SHORT).show()
        return
    }
    
    viewModel.addMoney(amount)
    
    Toast.makeText(context, "Добавлено $amount золотых", Toast.LENGTH_SHORT).show()
    
    etAddAmount.text.clear()
    layoutAddMoney.visibility = View.GONE
}
```

#### Метод 16: Сохранить перевод денег

Функция обрабатывает перевод денег между игроками или в банк:

```kotlin
private fun saveTransfer() {
    val amountStr = etRemoveAmount.text.toString()
    
    if (amountStr.isEmpty()) {
        Toast.makeText(context, "Введите сумму", Toast.LENGTH_SHORT).show()
        return
    }
    
    val amount = amountStr.toIntOrNull()
    if (amount == null || amount <= 0) {
        Toast.makeText(context, "Введите корректную сумму", Toast.LENGTH_SHORT).show()
        return
    }
    
    val recipientId = if (selectedRecipientIndex == 0) null else selectedRecipientIndex
    
    val success = viewModel.transferMoney(selectedPlayerId, recipientId, amount)
    
    if (success) {
        val recipientName = if (recipientId == null) "банк" else 
            viewModel.getPlayerNames().getOrNull(recipientId - 1) ?: "неизвестно"
        
        Toast.makeText(context, "Переведено $amount золотых в $recipientName", 
            Toast.LENGTH_SHORT).show()
        
        etRemoveAmount.text.clear()
        layoutRemoveMoney.visibility = View.GONE
    } else {
        Toast.makeText(context, "Недостаточно средств", Toast.LENGTH_SHORT).show()
    }
}
```

#### Метод 17: Настройка всех обработчиков кнопок

Эта функция настраивает обработчики для всех кнопок фрагмента:

```kotlin
private fun setupButtonListeners() {
    btnAddMoney.setOnClickListener {
        showAddMoneyForm()
    }
    
    btnRemoveMoney.setOnClickListener {
        showRemoveMoneyForm()
    }
    
    btnSaveAdd.setOnClickListener {
        saveAddMoney()
    }
    
    btnSaveTransfer.setOnClickListener {
        saveTransfer()
    }
    
}
```

Замените содержимое метода onCreateView вызовом созданных методов
```kotlin
	val view = inflater.inflate(R.layout.fragment_game, container, false)

	initializeViews(view)
	initializeGame()
	setupSpinnerListeners()
	setupObservers()
	setupButtonListeners()

	return view
```

---

## Шаг 11: Тестирование приложения

### Проверка функциональности:

1. **Создание игроков:**
   - Добавьте 2-4 игроков с разными именами
   - Проверьте, что нельзя добавить больше 4 игроков
   - Убедитесь, что кнопка "Начать игру" активируется только при 2+ игроках

2. **Навигация:**
   - Проверьте переход между фрагментами
   - Проверьте возврат назад к настройке

3. **Экран игры:**
   - Выберите игрока из списка
   - Проверьте отображение текущего баланса (должен быть 1500)
   - Нажмите "+" и добавьте деньги
   - Нажмите "-" и переведите деньги другому игроку
   - Переведите деньги в банк
   - Проверьте таблицу балансов всех игроков

4. **Граничные случаи:**
   - Попытка перевести больше, чем есть на балансе
   - Ввод некорректных сумм
   - Пустые поля ввода

---

## Заключение
-
-
-
PROFIT

Осталось раздобыть саму монополию
