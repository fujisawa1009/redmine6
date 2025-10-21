FROM redmine:6.0.6

RUN apt-get update
RUN apt-get -y install make
RUN apt-get -y install gcc

# Install Bundler 2.7.2 and set git safe.directory
RUN gem install bundler:2.7.2
RUN git config --global --add safe.directory /usr/src/redmine/plugins/redmine_github_hook
RUN git config --global --add safe.directory /project/fm-ptn-analysis.git
