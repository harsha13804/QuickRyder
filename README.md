# QuickRyder


sudo vi /usr/local/bin/watch-website.sh

#!/bin/bash

WATCH_DIR="/usr/share/nginx/html"

inotifywait -m -r \
  -e close_write,moved_to,create,delete \
  --format '%w%f' "$WATCH_DIR" |
while read FILE
do
    case "$FILE" in
        *.tmp|*.swp|*~) continue ;;
    esac
    echo "Website updated: $FILE"
    sleep 3
    aws lambda invoke \
      --function-name CloudFrontAutoInvalidation \
      --invocation-type Event \
      --cli-binary-format raw-in-base64-out \
      --payload '{}' \
      /tmp/lambda-response.json
    echo "CloudFront invalidation triggered."
done

sudo dnf install -y nginx inotify-tools

sudo vi /etc/systemd/system/website-watcher.service

[Unit]
Description=Nginx Website Auto Update Watcher
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/watch-website.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

sudo chmod +x /usr/local/bin/watch-website.sh

sudo systemctl daemon-reload

sudo systemctl enable --now website-watcher

sudo systemctl status website-watcher


#lambda Function

import boto3
import json
import uuid

cloudfront = boto3.client('cloudfront')

DISTRIBUTION_ID = "E38MGBC4XC51JD"

def lambda_handler(event, context):
    try:
        response = cloudfront.create_invalidation(
            DistributionId=DISTRIBUTION_ID,
            InvalidationBatch={
                'Paths': {
                    'Quantity': 1,
                    'Items': ['/*']
                },
                'CallerReference': str(uuid.uuid4())
            }
        )
        invalidation_id = response['Invalidation']['Id']
        print("CloudFront invalidation started")
        print("Invalidation ID:", invalidation_id)
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Invalidation created successfully',
                'id': invalidation_id
            })
        }
    except Exception as e:
        print("Invalidation failed:", str(e))
        raise

        raise




