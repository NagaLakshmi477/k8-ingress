FROM nginx

# Remove the default Nginx welcome page
RUN rm -rf /usr/share/nginx/html/index.html

# Create a new index.html file with APP 1content
RUN echo "<h1>Hi I am from APP 1</h1>" > /usr/share/nginx/html/index.html