## How to Clean Everything  
Sometimes you might run into really problematic issues or just want to experiment with the setup from scratch. Since everything is running on top of docker, this is really easy. Run the following to remove EVERYTHING and start from scratch.

```bash 
# See what stuff is avaialble 
docker images -a 
docker volume ls 

# Remove ALL the docker containers (Not just for fabric, ALL!!)
docker rm -vf $(docker ps -a -q) 

# .. and the images 
docker rmi -f $(docker images -a -q) 

# Volumes are not deleted by default 
docker volume ls 

# ... but we can get rid of those too 
docker system prune -a --volumes 

# Verify that everything is indeed gone 
docker volume ls 
docker images 
docker container ls 

# And just for final measure 
docker system prune --all 


# finally, get rid of the files as well. 
# Make sure you copy your code before doing this (if any)   
cd ~/fabric 
sudo rm -rf fabric-samples
```
