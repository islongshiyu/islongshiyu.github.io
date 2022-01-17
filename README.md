I use the `main` branch to store blog source codes and use the `gh-pages` branch for deploy.

i build this site with following steps:

1. Add github action configuration file `gh-pages.yml` to directory `.github/workflows`.
2. Modify the hexo theme `Next` config file `_config.next.yml`.
4. Add some posts and push the commits to the origin repo, it will auto build the site by github action.