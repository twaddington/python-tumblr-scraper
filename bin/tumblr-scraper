#!/usr/bin/python

import sys
import os
import time
import argparse
import urlparse
import json
import requests

from tumblr_scraper import __version__

def uprint(msg, newline=True):
    """
    Unbuffered print.
    """
    if not quiet:
        sys.stdout.write("%s%s" % (msg, "\n" if newline else ''))
        sys.stdout.flush()

def get_blog_url(name):
    return "http://{name}.tumblr.com/api/read/json".format(name=name)

def get_json_page(url, start=0, num=50):
    args = {}
    args['start'] = start
    args['num'] = num

    # Fetch the page
    r = requests.get(url, params=args)

    # Strip the jsonp garbage
    text = r.text.strip()[22:-1]

    # Return the result as a dict
    return json.loads(text)

def save_photo(s, url, out_dir):
    # Get the URL path
    url_path = urlparse.urlsplit(url).path

    # Get the filename from the URL path
    filename = os.path.basename(url_path)

    # Set our local out file
    photo_out = os.path.join(out_dir, filename)

    # Ensure the out directory exists
    try:
        os.makedirs(out_dir)
    except OSError:
        pass

    # Bail if this file already exists
    if os.path.isfile(photo_out):
        return
    else:
        resp = s.get(url, stream=True)
        if resp.ok:
            with open(photo_out, 'wb') as fd:
                for chunk in resp.iter_content(chunk_size=1024):
                    fd.write(chunk)

def save_post(json):
    # Get the GMT date components for this post
    year = json['date-gmt'][:4]
    month = json['date-gmt'][5:7]
    day = json['date-gmt'][8:10]

    # Find the out directory for this post
    out_dir = os.path.join(out, year, month, day, json['slug'])

    # INFO
    uprint(out_dir)

    # Create our HTTP session
    s = requests.Session()

    # Grab the post photo
    if 'photo-url-1280' in json:
        photo_url = json['photo-url-1280']

        # INFO
        uprint('    ' + photo_url)
        save_photo(s, photo_url, out_dir)

    # Grab all the photos
    if json['type'] == 'photo':
        for photo in json['photos']:
            photo_url = photo['photo-url-1280']

            # INFO
            uprint('    ' + photo_url)
            save_photo(s, photo_url, out_dir)

def main():
    # Create our parser
    global parser
    parser = argparse.ArgumentParser(prog='tumblr-scraper',
            description='Archives a Tumblr blog to a local directory')

    # Set up our command-line arguments
    parser.add_argument('blog_name')
    parser.add_argument('-o', '--out', default=os.getcwd(),
            help='set the out directory for the archive')
    parser.add_argument('--quiet', action='store_true',
            help='suppress all output except errors')
    parser.add_argument('--version', action='version',
            version='%(prog)s {v}'.format(v=__version__))

    # Get our arguments
    args = parser.parse_args()

    # Set our globals
    global quiet
    quiet = args.quiet

    global out
    out = os.path.expanduser(args.out)

    if not os.path.exists(out):
        parser.error('The out dir does not exist!')
    if not os.path.isdir(out):
        parser.error('The out dir is not a directory!')
    if not os.access(out, os.W_OK):
        parser.error('The out dir is not writable!')
    if not os.access(out, os.X_OK):
        parser.error('The out dir is not executable!')

    # Get our args
    blog_name = args.blog_name
    blog_url = get_blog_url(blog_name)

    # INFO
    uprint('Archiving \'%s\' to %s' % (blog_name, out,))

    # Get the total post count
    json = get_json_page(blog_url, 0, 0)
    total_count = json['posts-total']

    # INFO
    uprint('Total posts: %s' % (total_count,))

    # Start our elapsed timer
    start_time = time.time()

    # This is the main loop. We'll loop over each page of post
    # results until we've archived the entire blog.
    start = 0
    per_page = 50
    while start < total_count:
        json = get_json_page(blog_url, start)

        # Loop over each post in this batch
        for post in json['posts']:
            save_post(post)

        # Increment and grab the next batch of posts
        start += per_page

    # INFO
    minutes, seconds = divmod(time.time() - start_time, 60)
    uprint('Archived {count} posts in {m:02d}m and {s:02d}s'.format(\
            count=total_count, m=int(round(minutes)), s=int(round(seconds))))

if __name__ == '__main__':
    try:
        sys.exit(main())
    except KeyboardInterrupt:
        sys.exit()
