# Start from the base dockerfile image
FROM centos

# maintained by...
MAINTAINER Chris Ruffalo

# update yum
RUN yum update -y

# install packages
RUN yum install -y postfix

# remove yum bloat
RUN yum clean all

# copy script
COPY start.sh /start.sh

# copy configuration
COPY main.cf /etc/postfix/main.cf

# make runable
RUN chmod +x start.sh

# expose smtp port
EXPOSE 25

# start service
CMD ["/start.sh"]
