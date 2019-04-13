require "html-proofer"

task :default do
  sh("JEKYLL_ENV=production bundle exec jekyll build")
  puts "Jekyll successfully built"

  options = {
      # @todo Should try to enable this one
      # https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity
      check_sri: false,
      check_opengraph: true,
      check_favicon: true,
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
end
