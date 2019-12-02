
# Javascript Node CircleCI 2.0 configuration file
#
# Check https://circleci.com/docs/2.0/language-javascript/ for more details
#
version: 2
jobs:
  build:
    docker:
      # specify the version you desire here
      - image: starefossen/ruby-node:2-10

    steps:
      # Restoring source code cache
      - restore_cache:
          name: Restoring source code cache
          keys:
            - source-v1-{{ .Branch }}-{{ .Revision }}
            - source-v1-{{ .Branch }}-
            - source-v1-

      - checkout

      #Saving source code cache
      - save_cache:
          name: Saving source code cache
          key: source-v1-{{ .Branch }}-{{ .Revision }}
          paths:
            - ".git"

      #Restoring npm cache
      - restore_cache:
          name: Restoring npm cache
          keys:
            # when lock file changes, use increasingly general patterns to restore cache
            - npm-v1-{{ .Branch }}-{{ checksum "package-lock.json" }}
            - npm-v1-{{ .Branch }}-
            - npm-v1-

      # todo: find nicer solution than constantly "cd"ing to unit_testing
      - run: cd unit_testing && npm install

      #Saving npm cache
      - save_cache:
          name: Saving npm cache
          paths:
            - unit_testing/node_modules # location depends on npm version
          key: npm-v1-{{ .Branch }}-{{ checksum "package-lock.json" }}

      - run: cd unit_testing && npm test

      # Restoring bundler cache
      - restore_cache:
          name: Restoring bundler cache
          key: circlev2-{{ checksum "Gemfile.lock" }}

      #Bundler Dependencies
      - run:
          name: Jekyll deps
          command: bundle check --path=vendor/bundle || bundle install --path=vendor/bundle --jobs=4 --retry=3

      # Saving bundler cache
      - save_cache:
          name: Saving bundler cache
          key: circlev2-{{ checksum "Gemfile.lock" }}
          paths:
            - vendor/bundle

      - run:
          name: Build
          command: bundle exec jekyll build --verbose

      - run:
          name: Test
          command: bundle exec htmlproofer ./_site --assume-extension --check-html --disable-external --url-ignore "/#.*/"

  gist:
    docker:
      # specify the version you desire here
      - image: starefossen/ruby-node:2-10
    steps:
      - checkout

      #Restoring npm cache
      - restore_cache:
          name: Restoring npm cache
          keys:
            # when lock file changes, use increasingly general patterns to restore cache
            - npm-v1-{{ .Branch }}-{{ checksum "package-lock.json" }}
            - npm-v1-{{ .Branch }}-
            - npm-v1-

      # todo: find nicer solution than constantly "cd"ing to unit_testing
      - run: npm install

      #Saving npm cache
      - save_cache:
          name: Saving npm cache
          paths:
            - unit_testing/node_modules # location depends on npm version
          key: npm-v1-{{ .Branch }}-{{ checksum "package-lock.json" }}

      - run:
          name: "Uploading Coding Challenges Variations to GitHub Gists"
          command: "node checkVariations.js"
workflows:
  version: 2
  build-gist:
    jobs:
      - build
      - gist:
          requires:
            - build
          filters:
            branches:
              only: master
