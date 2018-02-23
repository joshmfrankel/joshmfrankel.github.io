require "html-proofer"

task :default do
  puts "Running CI tasks..."
  sh("JEKYLL_ENV=production bundle exec jekyll build")
  HTMLProofer.check_directory(
    "./_site",
    url_ignore: [/linkedin.com/] # Hidden behind auth wall
  ).run
  puts "Jekyll successfully built"
end
