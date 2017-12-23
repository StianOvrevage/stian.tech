## Setting up Jekyll locally on Windows

Start with a basic Ubuntu docker image with the GH-page repo bind-mounted to /blog

```
docker run -v D:\Data-Disk-Enc\git-github\StianOvrevage\stian.tech:/blog -p4000:4000 --rm -it ubuntu
```

Install some stuff in the container and start the dev-server.
```
apt-get update
apt-get install -y ruby-dev make gcc zlib1g-dev libcurl3 git
gem install bundler
bundle exec jekyll _3.3.0_ new blog
cd blog
bundle install
bundle update
bundle exec jekyll serve --host 0.0.0.0
```

Go to http://localhost:4000/ to preview.

### Tips

Locate UTF8-content in files: `grep --color='auto' -P -n "[\x80-\xFF]" *`

