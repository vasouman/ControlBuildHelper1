# ControlBuildHelper
A java Spring MVC application that automates the not very up to date build and deploy process we have at work for the "Control" project. While the build and deploy process is very custom and the app is definitely not meant to be easily, if at all, used with other projects, it has lot's of functionality that may come handy for all sorts of other tasks. The app is Spring MVC (no Maven), reads and writes Excell files using POI library, works with SVN repository using SVNkit, starts external executables and intercepts the console output, persists and restores its state using synchronization and interceptors, loads and saves property files, provides a log polling service.

Below are project description and installation instructions in Bulgarian:

Какво е "Адския билд тул на Васко"

	Java EE уеб приложение, предназначено да автоматизира процеса на прехвърляне на код между бранчовете, билдване и деплойване на проекта NRA Control. Състои се условно от две части:
	•Уизард за генериране на Release Instructions
	•Уизард за билдване на версия

	В по-голямата си част приложението изпълнява задачите, които билд инженерът изпълнява по време на билд, благодарение на което ръчният способ на билдване и генериране на RI остава в сила, а приложението имитира процеса в максимална степен. По всяко време билд процесът може да бъде прекъснат, върнат назад, подновен или продължен ръчно. 
	За повече информация за процеса на прехвърляне на код между бранчовете, билдване и деплойване в проекта NRA Control, прочети тук

Архитектура

	Приложението е реализирано посредством технологията Spring MVC. Фронт ендът е реализиран с jsp страници. Програмната логика е изцяло в бек енда, няма DB, но приложението използва различни ресурси на системата, на която върви (чете и пише на файловата система, стартира bat файлове (SVN command-line client, Ant...), манипулира работното копие на SVN, къмитва и ъпдейтва репото).

	Model

		Моделът се състои от няколко POJO и singleton stateless application scope сървиси (@Service), които се управляват от IOC контейнера на Спринг. Навсякъде, където се използват сървисите се инджектват (@Autowire).
		Единствено логинг сървисите са session scope, обвързани с контролерите, които също са session scope
		 Fun fact: Когато ползваш session scope, Spring MVC обвързва компонента/сървиса/контролера с нишката на HTTP заявката, което води до навъзможност да изполваш този компонент в друга нишка. Още се чудя как да го преборя това, защото ми трябва :( 

		Някои основни сървиси:
		•RiManagerService - чете и генерира екселски RI файлове. Използва POI библиотеката
		•SvnService - Комуникира с SVN репото и манипулира работното копие на машината, на която работи тула. Използва SVNkit. Не изпълнява write операции към репото, защото са твърде бавни. Комитваме като изпълняваме команди на SVN command-line client през CmdService
		•CmdService - стартира подадените му файлове/команди използвайки java.lang.ProcessBuilder и прихваща пренасочва лог съобщенията към логинг сървиса
		•PersistenceService - Прави autosave на контекста на билда на всяка съществена стъпка от процеса. Зарежда записания на файл контекст. Използва сериализация
		•PropertiesService - Чете и пише настройките на тула. Използва .property файлове и java.util.Properties
		•FileSystemService - създава директории, копита, мести и т.н. файлове
		•MyLogger - логинг сървис. Съхранява всевъзможни лог съобщения. Има възможност да ги записва на файл
		•LogPollerService - усъществява връзката между MyLogger и LogPoller контролера, който предоставя на клиента лог записите касаещи неговата сесия

		Стейта на уизардите се съхранява в контролерите, които са обвързани със сесията, в обекти от тип BuildWizardContext и RiBuilderContext съответно за "Уизард за билдване на версия" и "Уизард за генериране на Release Instructions" 

	View

		Вюто се състои от jsp страници, като единственото интересно тук, е че уизард логиката се реализира предимно в контролера, чрез сетване на модел атрибути в респонса, а jsp-тата ги усвояват по стандартизиран начин. Пример: Заглавието и подзаглавито на страницата; наличието и линка на бутоните "Напред", "Назад", "Прескочи"; дали да се вижда логинг конзолата и т.н.
		На страниците, на които е нужно да се визуализира логинг конзолата, върви един AJAX полер, който през няколко секунди взима от сървъра новите логове. Това е важно, тъй като броузъра и сървъра общуват синхронно и по време на дългите операции по билдване и синхронизация на SVN, потребителят трябва да вижда прогреса по задачите и евентуални съобщения за грешки

	Controller

		Контролера е реализиран чрез няколко session scoped Spring MVC контролер класове @Controller:
		•WizardController - отговаря за страниците принадлежащи към Уизарда за билдване на версия, както и общите странци, като тази. Основен клас в програмата - пази контекста на билда, със всички настройки и данни до момента, изпълнява билда викайки методите на сървисите
		•RiBuilderController - подобно на WizardController, но значително по-малък отговаря за страниците принадлежащи към Уизард за генериране на Release Instructions
		•LogPoller - отговаря на лог полинг заявките от вюто 
		•Филтри и интерцептори ?StatePersistInterceptor - сериализира и записва за диска контекста на билда на всяка значима страница от Уизарда за билдване на версия. Това позволява по всяко време да бъде продължен предишния билд, а при нужда и някой по-стар. Ако тула се ползва едновременно в две или повече сесии, не е ясно контекста на коя страница ще се запише като autosave. Въпреки това в билд папката на съответния билд ще се запише правилният контекст.
			?BuilderWizardContextInterceptor - следи дали контекста на билда (Уизарда за билдване на версия) е инициализиран за страниците, за които е необходим, и препраща към начална ако не е
			?RiBuilderContextInterceptor - следи дали контекста на генератора на Ri (Уизард за генериране на Release Instructions) е инициализиран за страниците, за които е необходим, и препраща към начална ако не е


Инсталиране

	Всички външни библиотеки използвани в приложението, както и библиотеките на Spring MVC са включени в EAR-а, което прави инсталирането му лесно. 
	Необходими са няколко допълнителни стъпки за успешното подкарване на приложението:
	•Създаване на работна папка на диска (пътят се настройва в hollymotherof.properties в WEB-INF), в която се намират: 
		?файла с настройките на приложението и различните видове билд - main.properties
		?темплейта за RI - RI/RI_Template_test.xlsx
		?папката RI\output, където се записват генерираните RI файлове

	•Инсталиране на SVN command-line client of Subversion
	•Настроки в сигурността на JRE, на което върви пролижението. В някои версии на джава SSL е ограничено до 1024b ключове, а за да работи SVNkit, трябва да разрешим повече. В този случай са необходими следните промени: 
		?Инсталиране на Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy
		?Добавяне на ред jdk.tls.disabledAlgorithms=DHE в jre\lib\security\java.security

	•Ако минаваме през прокси, настройваме проксито на SVN command-line clien: C:\Users\username\AppData\Roaming\Subversion\servers - сетваме клучовете: http-proxy-host и http-proxy-port
	•Checkout на бранчовете на проекта. Приложението използва всички бранчове на проекта, съществуващи от преди това ютилитите за билдване и други помощни тулове, както и модифицирани такива за неговите цели.
	•При първо пускане на билд от определен вид се налага настройка на пътища до ресурси - директории, файлове, bat файлове

