require "html-proofer"

task :default do
  puts "Running CI tasks..."
  sh("JEKYLL_ENV=production bundle exec jekyll build")
  HTMLProofer.check_directory(
    "./_site",
    url_ignore: [/linkedin.com|php-fig.org|bower.io|bost.ocks.org|elementary.io/] # Hidden behind auth wall
  ).run
  puts "Jekyll successfully built"
end


- ./_site/blog/How-to-configure-sublime-text-for-psr-standards/index.html
  *  External link http://www.php-fig.org failed: 301 SSL connect error
- ./_site/blog/gruntjs-or-how-i-learned-to-stop-worrying-and-automate-everything/index.html
  *  External link http://bower.io failed: 301 SSL connect error
- ./_site/blog/the-month-in-review-october-2015/index.html
  *  External link http://bost.ocks.org/mike/shuffle failed: 301 SSL connect error
  *  External link https://elementary.io failed: response code 0 means something's wrong.
             It's possible libcurl couldn't connect to the server or perhaps the request timed out.
             Sometimes, making too many requests at once also breaks things.
             Either way, the return message (if any) from the server is: SSL connect error
