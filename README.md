# cargo-catsをk8sを使わずに起動する手順

## 実行環境
- JDK17
- node18

## 手順
### git clone
```bash
git clone https://github.com/Contrast-Security-OSS/cargo-cats.git
```

### MySQL
```bash
docker run -d \
  --name mysql-cargocats \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=cargocats \
  -e MYSQL_DATABASE=db \
  -e MYSQL_USER=cargocats \
  -e MYSQL_PASSWORD=cargocats \
  mysql:8
```

### webhook service
*5001*ポート
```bash
cd services/webhookservice
pyenv local 3.9
pip install --no-cache-dir -r requirements.txt
pip install gunicorn
# Mac
sed -i '' 's/contrast-cargo-cats-db/localhost/g' ./app.py
# Amazon Linux 2023
sed -i 's/contrast-cargo-cats-db/localhost/g' ./app.py
# フォアグラウンド
gunicorn --bind 0.0.0.0:5001 --access-logfile - --error-logfile - --capture-output --log-level debug app:app
# バックグラウンド
nohup gunicorn --bind 0.0.0.0:5001 --access-logfile - --error-logfile - --capture-output --log-level debug app:app > gunicorn.log 2>&1 &
```

### image service
*80*ポート
```bash
cd services/imageservice
dotnet restore
dotnet publish -c Release -o out
# フォアグラウンド
dotnet ./out/imageservice.dll
# バックグラウンド
nohup dotnet ./out/imageservice.dll > imageservice.log 2>&1 &
```
**もしも下のようなエラーが出たら**
```bash
Framework: 'Microsoft.AspNetCore.App', version '8.0.0' (arm64)
.NET location: /usr/local/share/dotnet/

The following frameworks were found:
  10.0.0 at [/usr/local/share/dotnet/shared/Microsoft.AspNetCore.App]
```
**以下のコマンドを実行してから再度、dotnetコマンドを実行してください。**  
```bash
export DOTNET_ROLL_FORWARD=Major
```

### doc service
*5000*ポート
```bash
cd services/docservice
pyenv local 3.11
pip install --no-cache-dir -r requirements.txt
export FLASK_APP=app.py
export FLASK_ENV=development
export PYTHONUNBUFFERED=1
# フォアグラウンド
gunicorn --bind 0.0.0.0:5000 --workers 4 --timeout 120 app:app
# バックグラウンド
nohup gunicorn --bind 0.0.0.0:5000 --workers 4 --timeout 120 app:app > docservice.log 2>&1 &
```
※ gunicornの5000ポートでの起動でエラーになる場合、MacのAirPlayレシーバーが原因だったりするので  
Macのシステム設定で、AirPlayレシーバで設定を検索してオフにしてください。

### data service
*8080*ポート
```bash
cd services/dataservice
# Macの場合
sed -i '' 's/contrast-cargo-cats-db/localhost/g' ./src/main/resources/application.properties
# Amazon Linux 2023の場合
sed -i 's/contrast-cargo-cats-db/localhost/g' ./src/main/resources/application.properties
mvn clean package -Dmaven.test.skip=true
# フォアグラウンド
java -jar ./target/dataservice-0.0.1-SNAPSHOT.jar
# バックグラウンド
nohup java -jar ./target/dataservice-0.0.1-SNAPSHOT.jar > dataservice.log 2>&1 &
```

### label service
*3000*ポート
```bash
cd services/labelservice
npm install --production
# フォアグラウンド
npm start
# バックグラウンド
nohup npm start > labelservice.log 2>&1 &
```

### front service
*8081*ポート
```bash
cd services/frontgateservice
# Macの場合
sed -i '' 's|http://dataservice:8080|http://localhost:8080|g' ./src/main/resources/application.properties
sed -i '' 's|http://imageservice:80|http://localhost:80|g' ./src/main/resources/application.properties
sed -i '' 's|http://webhookservice:5000|http://localhost:5001|g' ./src/main/resources/application.properties
sed -i '' 's|http://labelservice:3000|http://localhost:3000|g' ./src/main/resources/application.properties
sed -i '' 's|http://docservice:5000|http://localhost:5000|g' ./src/main/resources/application.properties
# Amazon Linux 2023の場合
sed -i 's|http://dataservice:8080|http://localhost:8080|g' ./src/main/resources/application.properties
sed -i 's|http://imageservice:80|http://localhost:80|g' ./src/main/resources/application.properties
sed -i 's|http://webhookservice:5000|http://localhost:5001|g' ./src/main/resources/application.properties
sed -i 's|http://labelservice:3000|http://localhost:3000|g' ./src/main/resources/application.properties
sed -i 's|http://docservice:5000|http://localhost:5000|g' ./src/main/resources/application.properties
mvn clean package -Dmaven.test.skip=true
# フォアグラウンド
java -Dorg.apache.commons.collections.enableUnsafeSerialization=true -jar ./target/frontgateservice-0.0.1-SNAPSHOT.jar
# バックグラウンド
nohup java -Dorg.apache.commons.collections.enableUnsafeSerialization=true -jar ./target/frontgateservice-0.0.1-SNAPSHOT.jar > frontservice.log 2>&1 &
```

## 環境構築補足
### Amazon Linux 2023
#### node18
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc
nvm --version
nvm install 18
node -v
```

#### JDK17 & Maven
```bash
# 下のコマンドでjdk17の番号を選ぶ
alternatives --config java
alternatives --config javac
# Mavenも
dnf install -y maven
```
`mvn`コマンドを打つ前に`~/.bash_profile`に**JAVA_HOME**の設定も忘れずに。

#### Python3.9, 3.11
```bash
dnf install -y gcc zlib-devel bzip2 bzip2-devel readline-devel sqlite sqlite-devel openssl-devel tk-devel libffi-devel xz-devel
curl https://pyenv.run | bash
```
```bash
vim ~/.bash_profile
```
```
export PYENV_ROOT="$HOME/.pyenv"
command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
```
```bash
source ~/.bash_profile
pyenv install 3.9
pyenv install 3.11
```

#### dotnet
```bash
dnf install -y https://packages.microsoft.com/config/centos/8/packages-microsoft-prod.rpm
dnf install -y dotnet-sdk-8.0
dotnet --version
```

### Mac
#### node18
```bash
nvm install 18
```
#### Python3.9, 3.11
```bash
pyenv install 3.9
pyenv install 3.11
```

#### dotnet
https://dotnet.microsoft.com/en-us/download/dotnet/thank-you/sdk-10.0.100-macos-arm64-installer
https://dotnet.microsoft.com/en-us/download/dotnet/thank-you/runtime-8.0.22-macos-arm64-installer?cid=getdotnetcore
https://dotnet.microsoft.com/en-us/download/dotnet/thank-you/runtime-8.0.22-macos-arm64-installer?cid=getdotnetcore

いずれもDLしてインストールしてください。
```bash
# このコマンドでdotnetのランタイムが確認できます。
dotnet --list-runtimes
```
