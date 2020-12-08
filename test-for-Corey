The build subsystem

New file:  generate.py
#!/usr/bin/env python
# -*- coding: utf-8 -*-
from __future__ import print_function  # Python 2/3 compatibility
import boto3
import jinja2
import json
import base64
from jinja2_s3loader import S3loader
from boto3.dynamodb.conditions import Key
import dns.resolver

s3template_bucket = 'placeholder'
s3template_dir = 'templates'


def get_issue(foo=None, bar=None):
    answer = dns.resolver.resolve('issue.nextweekinaws.com','TXT')
    return list(answer.rrset.items.keys())[0].strings[0].decode("utf-8")


def get_extras_issue(foo=None, bar=None):
    answer = dns.resolver.resolve('extras-issue.nextweekinaws.com','TXT')
    return list(answer.rrset.items.keys())[0].strings[0].decode("utf-8")


def unfuck_link(href):
    # Ignore whitelisted items
    #    whitelist = [ 'youtube', 'news.ycombinator' ]
    #    if any(x in whitelist for x in href):
    #        return href
    # Is there UTM stuff? If so, discard it.
    try:
        href, _ = href.split('?', 1)
    except BaseException:
        pass
    # Gotta refuck the link with my own crap too!
    return href + "?utm_source=last_week_in_AWS&utm_medium=newsletter"


def build_link(link, issue, sponsored=False):
    desc = link.get('description')
    url = link.get('url')
    #    if sponsored == False:
    #        href = unfuck_link(href)
    extend = link.get('extended')
    payload = {"url": url, "issue": issue}
    preslug = json.dumps(payload).encode('utf-8')
    href = 'https://content.lastweekinaws.com/v1/' + str(base64.urlsafe_b64encode(preslug), 'utf-8')
    if not extend:
        extend = 'empty'
    # Got tired of copy-pasting stuff around. Let's automate it.
    if 'LINK' not in extend:
        built = "[%s](%s) - %s \n" % (desc, href, extend)
    else:
        link = '[%s](%s)' % (desc, href)
        extend = str.replace(extend, 'LINK', link)
        built = extend + "\n"
    return built


def build_tweet(link):
    href = link.get('url')
    tweet = "[Good Morning](" + href + ")!"
    return tweet


def get_approved_links(filter_tag):
    dynamodb = boto3.resource('dynamodb', region_name='us-west-2')
    table = dynamodb.Table('lwia-links')
    response = table.query(
        KeyConditionExpression=Key('issue').eq(int(filter_tag)))
    return response


def search(tags, links):
    results = [x for x in links if x.get("section", None) == tags]
    if results:
        return results
    return [element for element in links if tags in element]


def extras_handler(event, context):
    try:
        issue = event['path']['id']
    except:
        issue = get_extras_issue()
    links = get_approved_links(issue)
    sponsored = [x for x in links["Items"] if x.get("sponsored", None) == "1"]
    top = search("top", links['Items'])
    bottom = search("bottom", links['Items'])
    intro = search("intro", links['Items'])
    blogpost = search("content", links['Items'])
    title = search("title", links['Items'])
    top_blurbs = ''
    bottom_blurbs = ''
    issue_title = ''
    content_slug = ''
    introduction = ''
    for link in sponsored:
        if 'top' in link or link.get("section", None) == 'top':
            top_blurbs += "[[ sponsor ]] \n" + build_link(
                link, issue)
            top_blurbs += "[[ /sponsor ]] \n\n"
        if 'bottom' in link or link.get("section", None) == 'bottom':
            bottom_blurbs += "[[ sponsor ]] \n" + build_link(
                link, issue)
            bottom_blurbs += "[[ /sponsor ]] \n\n"
    for link in intro:
        introduction = link.get('extended')
    for link in blogpost:
        content_slug = link.get('extended')
    for link in title:
        issue_title = link.get('extended')

    # I can now update the skeleton copy as a template, instead of
    # embedded within python code. Harder to break it now.
    j2 = jinja2.Environment(loader=S3loader(s3template_bucket, s3template_dir))
    template = j2.get_template('extras-snarkdown.j2')

    markdown = template.render(issue=issue,
                               top=top_blurbs,
                               bottom=bottom_blurbs,
                               title=issue_title,
                               content=content_slug,
                               intro=introduction,
                               )
    return markdown


def lambda_handler(event, context):
    try:
        issue = event['path']['id']
    except:
        issue = get_issue()
    links = get_approved_links(issue)
    sponsored = [x for x in links["Items"] if x.get("sponsored", None) == "1"]
    community = search("community", links['Items'])
    aws = search("aws", links['Items'])
    tools = search("tools", links['Items'])
    jobs = search("jobs", links['Items'])
    tweet = search("tweet", links['Items'])
    intro = search("intro", links['Items'])
    title = search("title", links['Items'])
    aws_blurbs = ''
    sponsored_blurbs = ''
    community_blurbs = ''
    tools_blurbs = ''
    events_blurbs = ''
    jobs_blurbs = ''
    tweet_blurbs = ''
    issue_title = ''
    introduction = ''
    for link in sponsored:
        if 'community' in link or link.get("section", None) == 'community':
            sponsored_blurbs += "[[ from_the_community sponsored ]] \n" + build_link(
                link, issue)
            sponsored_blurbs += "[[ /from_the_community ]] \n\n"
            community.remove(link)
        if 'tools' in link or link.get("section", None) == 'tools':
            sponsored_blurbs += "[[ tool sponsored ]] \n" + build_link(
                link, issue)
            sponsored_blurbs += "[[ /tool ]] \n\n"
            tools.remove(link)
        if 'aws' in link or link.get("section", None) == 'aws':
            sponsored_blurbs += "[[ choice_cut sponsored ]] \n" + build_link(
                link, issue)
            sponsored_blurbs += "[[ /choice_cut ]] \n\n"
            aws.remove(link)
    for link in community:
        community_blurbs += "[[ from_the_community ]] \n" + build_link(
            link, issue)
        community_blurbs += "[[ /from_the_community ]] \n\n"
    for link in aws:
        aws_blurbs += "[[ choice_cut ]] \n" + build_link(
            link, issue) + "[[ /choice_cut ]] \n\n"
    for link in jobs:
        jobs_blurbs += "[[ job ]] \n" + build_link(
            link, issue, sponsored=True) + "[[ /job ]] \n\n"
    for link in tools:
        tools_blurbs += "[[ tool ]] \n" + build_link(
            link, issue) + "[[ /tool ]] \n\n"
    for link in tweet:
        tweet_blurbs += build_tweet(link)
    for link in intro:
        introduction = link.get('extended')
    for link in title:
        issue_title = link.get('extended')

    # I can now update the skeleton copy as a template, instead of
    # embedded within python code. Harder to break it now.
    j2 = jinja2.Environment(loader=S3loader(s3template_bucket, s3template_dir))
    template = j2.get_template('snarkdown.j2')

    markdown = template.render(issue=issue,
                               sponsored=sponsored_blurbs,
                               tools=tools_blurbs,
                               events=events_blurbs,
                               title=issue_title,
                               intro=introduction,
                               jobs=jobs_blurbs,
                               tweet=tweet_blurbs,
                               aws=aws_blurbs,
                               community=community_blurbs)
    return markdown


# Keep this around to handle output / invocation properly outside of the
# Lambda environment.
if __name__ == "__main__":
    print(lambda_handler('foo', 'bar'))
New file:  lockitin.py
#!/usr/bin/env python
# -*- coding: utf-8 -*-
from __future__ import print_function  # Python 2/3 compatibility
import boto3
import requests
import generate

s3template_bucket = 'placeholder'


def put_content(issue, data):
    s3_bucket = s3template_bucket
    key = issue + '.md'
    s3 = boto3.resource('s3')
    obj = s3.Object(s3_bucket, key)
    return obj.put(Body=data)


def get_content(issue=None):
    if issue:
        link = 'ugly' + issue
        if int(issue) > 10000:
            link = 'bad' + issue
    else:
        link = 'worse'
    content = requests.get(link)
    return content.text


def lambda_handler(event, context):
    print(event)
    try: 
        issue = event['path']['id']
    except:
        issue = generate.get_issue()
    data = get_content(issue)
    put_content(issue, data)
    return


# Keep this around to handle output / invocation properly outside of the
# Lambda environment.
if __name__ == "__main__":
    print(lambda_handler('foo', 'bar'))
New file:  podcast.py
#!/usr/bin/env python
# -*- coding: utf-8 -*-
from __future__ import print_function  # Python 2/3 compatibility
import boto3
import jinja2
from jinja2_s3loader import S3loader
import generate


s3template_bucket = 'placeholder'
s3template_dir = 'templates'


def lambda_handler(foo, bar):
    filter_tag = generate.get_issue()
    links = generate.get_approved_links(filter_tag)
    aws = generate.search("aws", links['Items'])
    aws_blurbs = ''

    for link in aws:
        aws_blurbs += "\n" + generate.build_link(link, filter_tag) + "\n"

    # I can now update the skeleton copy as a template, instead of
    # embedded within python code. Harder to break it now.
    j2 = jinja2.Environment(loader=S3loader(s3template_bucket, s3template_dir))
    template = j2.get_template('podcast.j2')

    markdown = template.render(issue=generate.get_issue(),
                               aws=aws_blurbs)
    return markdown


# Keep this around to handle output / invocation properly outside of the
# Lambda environment.
if __name__ == "__main__":
    print(lambda_handler('foo', 'bar'))
New file:  retag.py
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import sys
import boto3
from generate import get_issue, get_extras_issue

database_zone = 'Z2S7VHPXUV19LK'


def set_extras_issue(event=None, context=None, tag=None):
    old = get_extras_issue()
    new = "\"" + str(int(old) + 1) + "\""
    client = boto3.client('route53')
    response = client.change_resource_record_sets(
        HostedZoneId=database_zone,
        ChangeBatch={
            "Comment": "Incrementing issue",
            "Changes": [
                {
                    "Action": "UPSERT",
                    "ResourceRecordSet": {
                        "Name": 'extras-issue.nextweekinaws.com',
                        "Type": "TXT",
                        "TTL": 5,
                        "ResourceRecords": [
                            {
                                "Value": new
                            },
                        ],
                    }
                },
            ]
        })
    return new


def set_issue(event=None, context=None, tag=None):
    old = get_issue()
    new = "\"" + str(int(old) + 1) + "\""
    client = boto3.client('route53')
    response = client.change_resource_record_sets(
        HostedZoneId=database_zone,
        ChangeBatch={
            "Comment": "Incrementing issue",
            "Changes": [
                {
                    "Action": "UPSERT",
                    "ResourceRecordSet": {
                        "Name": 'issue.nextweekinaws.com',
                        "Type": "TXT",
                        "TTL": 5,
                        "ResourceRecords": [
                            {
                                "Value": new
                            },
                        ],
                    }
                },
            ]
        })
    return new


def handler(event, context):
    set_issue()


if __name__ == '__main__':
    tag = sys.argv[1]
    retag = set_issue(tag)
    bump = retag_links(tag)
New file:  sponsor.py
import json
import logging
import os
import time
import uuid
from datetime import datetime

import boto3
dynamodb = boto3.resource('dynamodb')


def create(event, context):
    data = json.loads(event['body'])
    if 'text' not in data:
        logging.error("Validation Failed")
        raise Exception("Couldn't create the link.")
    
    timestamp = str(datetime.utcnow().timestamp())

    table = dynamodb.Table(os.environ['DYNAMODB_TABLE'])

    item = {
        'id': data['issue'],
        'content': data['content'],
        'description': data['description'],
        'createdAt': timestamp,
        'updatedAt': timestamp,
    }

    # write the todo to the database
    table.put_item(Item=item)

    # create a response
    response = {
        "statusCode": 200,
        "body": json.dumps(item)
    }

    return response
