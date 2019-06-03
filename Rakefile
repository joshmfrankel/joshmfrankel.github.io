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
      # Regex for my site fixes the issue where a post hasn't been
      # published yet but the seo metadata is pointing at the live url.
      # This causes a 404 until the post is published. Regex below is
      # constrained tighly to this scenario.
      url_ignore: [/globalnerdy.com.*|http:\/\/joshfrankel.me\/blog\/.*\/$/],
      cache: {
        # Cache external url checking for 6 weeks
        timeframe: '6w'
      }
    }

  HTMLProofer.check_directory(
    "./_site",
    options
  ).run
end
