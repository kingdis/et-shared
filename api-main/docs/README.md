# Effitime API

## Development

### Run the application in docker
#### Test Dockerfile
```bash
docker build .
```

#### Run docker compose
##### Run mongo only
```bash
docker compose --profile mongo up
```

##### Run all services
```bash
docker compose --profile all up
```

### Run the application in local microk8s
1. Merege the latest development branch (develop) to the stage branch. This will trigger the GitHub auto build action workflow to build the docker image and push to the ghcr.io/kingdis/effitime-api.
2. Go to the actions page, find the image tag of the latest workflow run.
3. Update the image tag in effitime-ops/app-chart/values.yaml.
4. Go to the wsl, navigate to the effitime-ops directory, run `./install.sh`.

### Run the application in Hetzner Cloud
1. Merege the latest development branch (develop) to the stage branch. This will trigger the GitHub auto build action workflow to build the docker image and push to the ghcr.io/kingdis/effitime-api.
2. Go to the actions page, find the image tag of the latest workflow run.
3. Update the image tag in effitime-ops/app-chart/values.yaml.
4. Connect to the Hetzner Clould using SSH, login to the server using kingdis account.
5. Pull the effitime-ops repository, navigate to the effitime-ops directory, run `./install.sh`.


## Temporary commands

### Update all statistics by aspect and period
After logging in to the app, run the following url in the browser's address bar.
```
GET http://localhost:3999/api/statistics/aspect-duration/update-all
```
or
```
GET https://api.effetime.com/api/statistics/aspect-duration/update-all
```

### Update activity category usage frequency
```
GET http://localhost:3999/api/user/profile/update-activity-category-usage-frequency
```
or
```
GET https://api.effetime.com/api/user/profile/update-activity-category-usage-frequency
```
