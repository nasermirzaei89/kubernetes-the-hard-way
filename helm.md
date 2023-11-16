# Helm

## Install Helm


https://helm.sh/docs/intro/install/#from-apt-debianubuntu

```shell
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## Enable Completion

https://helm.sh/docs/helm/helm_completion_bash/

For Linux:

```shell
helm completion bash > /etc/bash_completion.d/helm
```