# Get Started

## Table of content

Add repository

```
# add the repository
sudo tee /etc/apt/sources.list.d/pgdg.list <<END
deb http://apt.postgresql.org/pub/repos/apt/ bionic-pgdg main
END

# get the signing key and import it
wget https://www.postgresql.org/media/keys/ACCC4CF8.asc
sudo apt-key add ACCC4CF8.asc

# fetch the metadata from the new repo
sudo apt-get update
```

Install postgres

```
sudo apt-get -y install postgresql-11 postgresql-contrib-11
```

Install pg_partman

```
sudo apt-get -y install postgresql-11-partman
```

Download test data from github archive

```
wget https://data.gharchive.org/2015-01-01-15.json.gz

gunzip 2015-01-01-15.json.gz
```

Install Jq for prettier format on console

```
sudo apt-get -y install jq
```

Explore data

```
cat 2015-01-01-15.json jq
```
Results should be
```
.......
{
    "id": "2489678823",
    "type": "CreateEvent",
    "actor": {
      "id": 9666449,
      "login": "automatic-frog",
      "gravatar_id": "",
      "url": "https://api.github.com/users/automatic-frog",
      "avatar_url": "https://avatars.githubusercontent.com/u/9666449?"
    },
    "repo": {
      "id": 26456747,
      "name": "osp/osp.relearn.off-grid",
      "url": "https://api.github.com/repos/osp/osp.relearn.off-grid"
    },
    "payload": {
      "ref": "master",
      "ref_type": "branch",
      "master_branch": "master",
      "description": "MIRROR of http://osp.kitchen/relearn/off-grid",
      "pusher_type": "user"
    },
    "public": true,
    "created_at": "2015-01-01T15:59:56Z",
    "org": {
      "id": 9216151,
      "login": "osp",
      "gravatar_id": "",
      "url": "https://api.github.com/orgs/osp",
      "avatar_url": "https://avatars.githubusercontent.com/u/9216151?"
    }
  }
.........
```
