Для публикации нового поста:
1. Пишем пост
2. На WSL dыполняем билд и копирование
```bash
JEKYLL_ENV=production bundle exec jekyll build
scp -r -i ~/.ssh/id_ed25519 _site/ stitov@setigate.ru:~/
```
3. На ВМ выполняем скрипт install_site.sh