# CI/CD для статичного сайту: Git → GitHub → CodePipeline → CodeBuild → S3

## Вступ і архітектура

Підсумок: створюємо пайплайн, де `git push` у GitHub автоматично перевіряє HTML у CodeBuild і публікує сайт «Шпаргалка мережевика» в S3 з публічним доступом по HTTP.

**Важливе уточнення щодо CodeDeploy.** CodeDeploy не вміє деплоїти в S3. Його цілі: EC2/on-premises, Lambda, ECS. Для S3 у CodePipeline є вбудована дія Deploy → Amazon S3. Тому основний пайплайн (кроки 1-7): GitHub → CodePipeline → CodeBuild (перевірка) → S3 Deploy. 

**Потік:** локальний git → push → GitHub → CodePipeline (Source → Build → Deploy) → S3 Static Website → публічний інтернет.

**Припущення:**

- регіон `eu-central-1` (Франкфурт) або будь-який інший, але один для бакета, Connection, CodeBuild і пайплайна; у перевірці лабораторії використано eu-north-1 (Стокгольм), тоді endpoint сайту має вигляд ...s3-website.eu-north-1.amazonaws.com;
- навчальний AWS-акаунт із правами адміністратора та акаунт GitHub;
- у нас є  Windows/LinuxMacOS, Git і термінал; назви `<ІМ'Я>` потрібно замінити на свої.

## Крок 1. Локальний репозиторій і зміст сайту

Зміст сайту — «Шпаргалка мережевика»: калькулятор підмереж (CIDR) на JavaScript і таблиця команд Linux для діагностики мережі. Це близько до того, що студенти вже знають.

Налаштуйте git і створіть репозиторій:

```bash
git config --global user.name  "Ваше Імʼя"
git config --global user.email "you@example.com"

mkdir netcheat && cd netcheat
git init -b main
```

Не використовуйте заглушки на кшталт `student@gmail.com`: GitHub зіставляє автора коміту за email і може приписати коміт чужому акаунту, який має цю адресу. Вкажіть власне імʼя та email акаунта GitHub або noreply-адресу (GitHub → Settings → Emails, вигляд `12345678+логін@users.noreply.github.com`). Перевірте налаштування командою `git config --global --list` до першого коміту.

Файл `index.html`:

```html
<!DOCTYPE html>
<html lang="uk">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Шпаргалка мережевика</title>
  <style>
    body{font-family:system-ui,sans-serif;max-width:800px;margin:2rem auto;padding:0 1rem;background:#0f172a;color:#e2e8f0}
    h1,h2{color:#38bdf8}
    input,button{padding:.5rem;font-size:1rem;border-radius:6px;border:1px solid #475569}
    button{background:#38bdf8;border:none;cursor:pointer}
    pre{background:#1e293b;padding:1rem;border-radius:8px;overflow-x:auto}
    table{border-collapse:collapse;width:100%}
    td,th{border:1px solid #334155;padding:.4rem;text-align:left}
    code{color:#a7f3d0}
  </style>
</head>
<body>
  <h1>Шпаргалка мережевика</h1>
  <p>Цей сайт розгорнуто автоматично через CI/CD пайплайн AWS. Версія збірки: <b>v1</b></p>

  <h2>Калькулятор підмереж</h2>
  <input id="cidr" value="192.168.10.77/26" size="22">
  <button onclick="calc()">Порахувати</button>
  <pre id="out">Введіть IP/префікс, наприклад 10.0.0.5/24</pre>

  <h2>Команди Linux для діагностики мережі</h2>
  <table>
    <tr><th>Завдання</th><th>Команда</th></tr>
    <tr><td>IP-адреси інтерфейсів</td><td><code>ip -br a</code></td></tr>
    <tr><td>Таблиця маршрутів</td><td><code>ip route</code></td></tr>
    <tr><td>Відкриті порти та процеси</td><td><code>ss -tulpn</code></td></tr>
    <tr><td>Шлях пакета до хоста</td><td><code>traceroute / mtr example.com</code></td></tr>
    <tr><td>DNS-запит</td><td><code>dig +short example.com</code></td></tr>
    <tr><td>Перевірка HTTP-заголовків</td><td><code>curl -I https://example.com</code></td></tr>
    <tr><td>Захоплення трафіку</td><td><code>sudo tcpdump -i eth0 port 80</code></td></tr>
    <tr><td>Перевірка порту</td><td><code>nc -zv host 22</code></td></tr>
  </table>

  <script>
    function toInt(ip){return ip.split('.').reduce((a,o)=>(a*256)+Number(o),0)>>>0}
    function toIp(n){return [24,16,8,0].map(s=>(n>>>s)&255).join('.')}
    function calc(){
      const out=document.getElementById('out');
      const m=document.getElementById('cidr').value.trim()
        .match(/^(\d{1,3}(?:\.\d{1,3}){3})\/(\d{1,2})$/);
      if(!m){out.textContent='Формат: 192.168.1.10/24';return}
      const octets=m[1].split('.').map(Number), p=Number(m[2]);
      if(p>32||octets.some(o=>o>255)){out.textContent='Некоректне значення';return}
      const ip=toInt(m[1]);
      const mask=p===0?0:(0xFFFFFFFF<<(32-p))>>>0;
      const net=(ip&mask)>>>0;
      const bc=(net|(~mask>>>0))>>>0;
      const hosts=p>=31?(p===32?1:2):Math.pow(2,32-p)-2;
      out.textContent=
        `Мережа:     ${toIp(net)}/${p}\n`+
        `Маска:      ${toIp(mask)}\n`+
        `Broadcast:  ${toIp(bc)}\n`+
        `Перший хост:${toIp(p>=31?net:net+1)}\n`+
        `Останній:   ${toIp(p>=31?bc:bc-1)}\n`+
        `Хостів:     ${hosts}`;
    }
    calc();
  </script>
</body>
</html>
```

Файл `error.html` (сторінка помилки 404):

```html
<!DOCTYPE html>
<html lang="uk"><head><meta charset="utf-8"><title>404</title></head>
<body style="font-family:sans-serif;text-align:center;margin-top:20vh">
<h1>404</h1><p>Пакет загубився десь між маршрутизаторами.</p>
</body></html>
```

Файл `buildspec.yml` — інструкція для CodeBuild:

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 20
    commands:
      - npm install -g htmlhint
  build:
    commands:
      - echo "Перевірка HTML..."
      - htmlhint index.html error.html
  post_build:
    commands:
      - echo "Збірка успішна $(date)"

artifacts:
  files:
    - index.html
    - error.html
```

Зробіть перший коміт і перевірте сайт локально:

```bash
git add .
git commit -m "Initial commit: netcheat static site"
python3 -m http.server 8000   # відкрийте http://localhost:8000
```

## Крок 2. GitHub

Локальний репозиторій потрібно запушити у порожній репозиторій на GitHub.

1. На github.com створіть **порожній** репозиторій `netcheat` (без README, `.gitignore` і ліцензії).
2. Привʼяжіть віддалений репозиторій і запуште:

```bash
git remote add origin git@github.com:<ІМ'Я>/netcheat.git   # або https://github.com/<ІМ'Я>/netcheat.git
git push -u origin main
```

Для SSH-доступу згенеруйте ключ командою `ssh-keygen -t ed25519`, додайте вміст `~/.ssh/id_ed25519.pub` у GitHub → Settings → SSH and GPG keys. Альтернатива: HTTPS із Personal Access Token.

## Крок 3. S3-бакет для сайту

S3-бакет зі статичним хостингом і публічним читанням дає сайт, доступний з інтернету по HTTP.

**Створення бакета** (консоль AWS → S3 → Create bucket):

1. Назва глобально унікальна, наприклад `netcheat-<ІМ'Я>-2026`. Регіон: `eu-central-1`.
2. **Block Public Access:** зніміть галочку *Block all public access* і підтвердіть, що розумієте ризик. Для навчального сайту це допустимо.
3. Натисніть *Create bucket*.

**Статичний хостинг:** бакет → *Properties* → внизу *Static website hosting* → Edit → Enable:

- Index document: `index.html`
- Error document: `error.html`

Запишіть **Bucket website endpoint**, наприклад `http://netcheat-<ІМ'Я>-2026.s3-website.eu-central-1.amazonaws.com`.

**Bucket policy** (*Permissions → Bucket policy*):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicRead",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::netcheat-<ІМ'Я>-2026/*"
  }]
}
```

S3 website endpoint працює лише по **HTTP**. Для HTTPS потрібен CloudFront.

## Крок 4. GitHub Connection (AWS CodeConnections)

Connection — це ресурс AWS, який дає CodePipeline право читати ваш GitHub-репозиторій і отримувати події push. Він працює через GitHub App (*AWS Connector for GitHub*), а не через персональні токени: паролі не зберігаються, доступ обмежується конкретними репозиторіями й відкликається в будь-який момент. Сервіс раніше називався *CodeStar Connections*, тому в старих інструкціях і в ARN можна побачити `codestar-connections`; суть та сама.

**Передумови**

- Ви в тому ж **регіоні**, де буде пайплайн (`eu-central-1`): Connection привʼязаний до регіону.
- Репозиторій `netcheat` існує на GitHub, код запушено.
- Ви власник акаунта GitHub із репозиторієм. Якщо репозиторій в організації, потрібні права owner для встановлення застосунку.

**Варіант А. Створення з майстра пайплайна (найпростіше)**

1. У майстрі *Create pipeline* на кроці **Source** виберіть provider **GitHub (via GitHub App)**.
2. Натисніть **Connect to GitHub** і введіть **Connection name**, наприклад `github-netcheat`.
3. Натисніть **Connect to GitHub** ще раз. Відкриється спливаюче вікно GitHub (дозвольте popup у браузері, якщо його заблоковано).
4. Увійдіть у GitHub і натисніть **Authorize AWS Connector for GitHub**.
5. У вікні AWS біля поля *GitHub Apps* натисніть **Install a new app**. Авторизація дає застосунку ідентифікацію, а встановлення дає доступ до репозиторіїв.
6. На GitHub виберіть акаунт або організацію й доступ: **Only select repositories** → `netcheat` (мінімальні привілеї) або *All repositories*.
7. Натисніть **Install** (за потреби підтвердіть паролем).
8. Вікно закриється, у полі *GitHub Apps* зʼявиться ідентифікатор встановлення. Натисніть **Connect**.
9. У майстрі в полі *Connection* буде вибрано ваш конект із повідомленням «Ready to connect».
10. Виберіть **Repository name** `<ІМ'Я>/netcheat` і **Branch** `main`.
11. Залиште ✅ **Start the pipeline on source code change** і формат артефакту **CodePipeline default**.

**Варіант Б. Створення заздалегідь (зручно для пояснення студентам)**

1. AWS Console → **Developer Tools** → **Settings** → **Connections** → **Create connection**.
2. Provider **GitHub**, імʼя, **Connect to GitHub**, далі як у пунктах 3-8 варіанта А.
3. Статус конекту має бути **Available**. Якщо **Pending**, не завершено встановлення застосунку: відкрийте конект і натисніть **Update pending connection**.
4. У майстрі пайплайна виберіть цей конект зі списку.

**Права IAM.** Майстер сам додає дозвіл до ролі `AWSCodePipelineServiceRole-...`. Якщо Source stage не працює, перевірте наявність:

```json
{
  "Effect": "Allow",
  "Action": "codeconnections:UseConnection",
  "Resource": "arn:aws:codeconnections:eu-central-1:<ACCOUNT_ID>:connection/<ID>"
}
```

У старих ролях може бути `codestar-connections:UseConnection` із відповідним ARN; обидва варіанти працюють.

**Перевірка, що все працює**

1. GitHub → Settings → Applications → Installed GitHub Apps: *AWS Connector for GitHub* має мати доступ до `netcheat`. Тут же його можна відкликати.
2. В AWS статус конекту **Available**.
3. Тестовий `git push` у `main` запускає новий запуск пайплайна зі статусом *In progress*; у Source stage видно ваш коміт.

**Типові проблеми з Connection**

| Симптом | Причина і рішення |
| --- | --- |
| Спливаюче вікно не відкривається | Браузер блокує popup. Дозвольте для `console.aws.amazon.com` |
| Статус конекту Pending | Не завершено встановлення застосунку. Відкрийте конект → Update pending connection |
| У списку немає репозиторію `netcheat` | Застосунок встановлено без доступу до нього. GitHub: Settings → Applications → AWS Connector for GitHub → Configure → додайте репозиторій |
| Репозиторій в організації, встановлення «чекає схвалення» | Ви не owner. Власник схвалює запит у Settings → GitHub Apps → Pending requests |
| Конект є, але не видно в майстрі | Різні регіони. Створіть конект у тому ж регіоні, що й пайплайн |
| Пайплайн стартує один раз, на push не реагує | Вимкнено автозапуск, не та гілка, або trigger filters у V2-пайплайні не збігаються з гілкою |
| `Unable to use Connection` / AccessDenied у Source | Ролі пайплайна бракує `UseConnection` на ARN конекту |
| Конект «зламався» після змін на GitHub | Перевстановіть застосунок, оновіть конект. Якщо не допомагає, створіть новий і виберіть його в Source stage |

**Примітки.** Один конект можна використовувати для багатьох репозиторіїв і пайплайнів, тож на курсі достатньо створити його раз. Формат артефакту *Full clone* потрібен, лише якщо CodeBuild виконує `git`-команди; тоді ролі CodeBuild додатково потрібен `codeconnections:UseConnection`. Для цієї лабораторної він не потрібен.

## Крок 5. CodeBuild-проєкт

CodeBuild виконує `buildspec.yml`: ставить htmlhint і перевіряє HTML. Якщо перевірка падає, пайплайн зупиняється до Deploy.

CodeBuild → Create build project:

| Поле | Значення |
| --- | --- |
| Project name | `netcheat-build` |
| Source | No source (джерело передасть CodePipeline) |
| Environment | Managed image, Amazon Linux, Standard, `aws/codebuild/amazonlinux-x86_64-standard:5.0` |
| Service role | New service role (створюється автоматично) |
| Buildspec | Use a buildspec file (`buildspec.yml` у корені) |
| Artifacts | No artifacts (артефактами керує пайплайн) |
| CloudWatch logs | Увімкнено |

Натисніть *Create build project*.

## Крок 6. CodePipeline

Пайплайн з трьох стадій (Source → Build → Deploy) запускається автоматично від кожного push у `main`.

CodePipeline → Create pipeline → Build custom pipeline:

1. **Settings:** Name `netcheat-pipeline`, Service role: New service role, Execution mode: Queued або Superseded.
2. **Source stage:** provider **GitHub (via GitHub App)** → вибрати Connection із кроку 4 → репозиторій `netcheat`, гілка `main`. Залиште увімкненим *Start the pipeline on source code change*; Output artifact format: *CodePipeline default*.
3. **Build stage:** виберіть **Other build providers** → **AWS CodeBuild** (варіант *Commands* не підходить: він виконує shell-команди без `buildspec.yml`). Проєкт `netcheat-build`, Input artifact `SourceArtifact`, Build type: Single build. Зніміть галочку *Enable automatic retry on stage failure*, щоб збірка з помилкою в HTML не повторювалась і демонстрація не затягувалась.
4. **Test stage:** Skip.
5. **Deploy stage:** provider **Amazon S3**; Region такий самий, як у бакета; Input artifact `BuildArtifact`; Bucket із кроку 3; поставте ✅ **Extract file before deploy** (інакше в бакет потрапить zip). Поле *S3 object key* після цього зникає, а *Deployment path* залиште порожнім: інакше файли опиняться в підпапці й сайт не відкриється за кореневим URL. Зніміть галочку *Configure automatic rollback on stage failure*, щоб студенти бачили реальну поведінку пайплайна.
6. Натисніть *Create pipeline*. Він запуститься одразу.

Якщо Deploy падає з `AccessDenied`, додайте ролі пайплайна (`AWSCodePipelineServiceRole-...`) політику на ваш бакет з діями `s3:PutObject`, `s3:GetObject`, `s3:ListBucket`, `s3:DeleteObject`, `s3:PutObjectAcl`.

## Крок 7. Перевірка та демонстрація

Головна ідея: зламаний коміт не доходить до продакшену.

1. Відкрийте website endpoint з кроку 3: сайт має завантажитись.
2. Змініть версію в `index.html` і запуште:

```bash
sed -i 's/v1/v2/' index.html
git commit -am "bump to v2" && git push
```

3. Пайплайн стартує сам. За 1-2 хвилини оновіть сторінку: зʼявиться `v2`.
4. **Завдання «зламай збірку».** Видаліть закриваючий тег (наприклад `</table>`) і запуште. CodeBuild відхилить зміни через htmlhint, стадія Deploy не виконається, на проді лишається робоча версія.
5. **Завдання на розширення:**
   - додайте у `buildspec.yml` перевірку, що `index.html` містить рядок `Шпаргалка` (наприклад `grep -q "Шпаргалка" index.html`);
   - додайте нову команду в таблицю;
   - додайте перед Deploy дію *Manual approval*, щоб публікація потребувала схвалення.


## Прибирання та типові проблеми

Після лабораторної видаліть ресурси, щоб уникнути витрат: пайплайн, CodeBuild-проєкт, артефактний бакет `codepipeline-<регіон>-...` (спочатку очистіть його), бакет із сайтом, Connection (Developer Tools → Settings → Connections → Delete) і створені IAM-ролі. За бажанням деінсталюйте *AWS Connector for GitHub* у налаштуваннях GitHub.

| Симптом | Причина |
| --- | --- |
| 403 Forbidden на сайті | Не знято Block Public Access або немає bucket policy |
| Скачується zip замість сайту | Не увімкнено *Extract file before deploy* |
| Пайплайн не стартує від push | Не авторизовано GitHub App або неправильна гілка |
| Build падає на `htmlhint` | Помилка в HTML; дивіться CloudWatch logs білда |
| Deploy: AccessDenied | Ролі пайплайна бракує прав на бакет (див. крок 6) |
