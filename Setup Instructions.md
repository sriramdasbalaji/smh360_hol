1.	Create Ubuntu 18.04LTS from Azure.
2.	Install Desktop (Optional)
    ``` 
    sudo apt-get update
    sudo apt-get install xfce4
    ```
3.	Installed xrdp (Optional)
    ```
    sudo apt-get install xrdp
    echo xfce4-session >~/.xsession
    sudo service xrdp restart
    ```
4.	Chrome (optional)
 ```
 wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | sudo apt-key add -
echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" | sudo tee /etc/apt/sources.list.d/google-chrome.list
sudo apt-get update
sudo apt-get -y install google-chrome-stable
```
5.	SublimeText (Optional)
```
wget -qO - https://download.sublimetext.com/sublimehq-pub.gpg | sudo apt-key add -

echo "deb https://download.sublimetext.com/ apt/stable/" | sudo tee /etc/apt/sources.list.d/sublime-text.list
sudo apt-get update
sudo apt-get install sublime-text
```
6.	Install Docker
https://docs.docker.com/v17.09/engine/installation/linux/docker-ce/ubuntu/#set-up-the-repository

7.	Set sudo user for current session `sudo -s`

8.	Install Azure CLI
https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-apt?view=azure-cli-latest

9.	Install kubectl

     `sudo snap install kubectl --classic`

10.	Dockercompose 

    `sudo apt install docker-compose` 

11.	If you face error Remote error from secret service: org.freedesktop.DBus.Error.ServiceUnknown: The name org.freedesktop.secrets was not provided by any .service files
Error saving credentials: error storing credentials - err: exit status 1, out: `The name org.freedesktop.secrets was not provided by any .service files`
Follow  https://stackoverflow.com/questions/50151833/cannot-login-to-docker-account


 

 

