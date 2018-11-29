require "html-proofer"

task :default do
  puts "Running CI tasks..."
  sh("JEKYLL_ENV=production bundle exec jekyll build")

  options = {
      # @todo Should try to enable this one
      # https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity
      check_sri: false,
      check_opengraph: true,
      check_favicon: false, # @todo I should add a favicon
      http_status_ignore: [999],
      url_ignore: [/globalnerdy.com.*/],
      cache: {
        timeframe: '6w'
      }
    }

  HTMLProofer.check_directory(
    "./_site",
    options
  ).run
  puts "Jekyll successfully built"
end
