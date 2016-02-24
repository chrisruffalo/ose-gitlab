# Start from the base dockerfile image
FROM gitlab/gitlab-ce:8.5.1-ce.0

# maintained by...
MAINTAINER Chris Ruffalo

# copy patched gitlab-shell.rb
RUN	curl https://gitlab.com/chrisruffalo/omnibus-gitlab/raw/master/files/gitlab-cookbooks/gitlab/recipes/gitlab-shell.rb > /opt/gitlab/embedded/cookbooks/gitlab/recipes/gitlab-shell.rb

# ======= use the same entrypoint and metadata from the original ======= 

# Allow to access embedded tools
ENV PATH /opt/gitlab/embedded/bin:/opt/gitlab/bin:/assets:$PATH

# Resolve error: TERM environment variable not set.
ENV TERM xterm

# Expose web & ssh
EXPOSE 443 80 22

# Define data volumes
VOLUME ["/etc/gitlab", "/var/opt/gitlab", "/var/log/gitlab"]

# Wrapper to handle signal, trigger runit and reconfigure GitLab
CMD ["/assets/wrapper"]
