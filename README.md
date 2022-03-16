# jofem48-repo-pbl
#Jude Ofem Projects
#Jude, [3/16/2022 8:45 AM]

import boto3
import json
import datetime
import csv
import re
import argparse
from botocore.exceptions import ClientError
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.application import MIMEApplication


# Set the global variables
globalVars  = {}
globalVars['Owner']                 = "Babatunde"
globalVars['Environment']           = "Test"
globalVars['REGION_NAME']           = "us-east-2"
globalVars['tagName']               = "EBS-Volume-Clean-Up-Test"
globalVars['findNeedle']            = "Name"
globalVars['tagsToInclude']         = "DO NOT DELETE"
ec2       = boto3.resource('ec2', region_name = globalVars['REGION_NAME'] )
snsClient = boto3.client('sns','us-east-2')

#time = datetime.datetime.now().strftime ('%Y-%m-%d-%H-%M-%S')
#s3 = boto3.resource('s3')
#bucket = s3.Bucket('demo-s3-unused-ebs')
#file_name = ('backup_job_weekly_report_' + '.csv')
#s3_path = 'Inventory/Weekly/' + file_name


lambda_client = boto3.client('lambda')

s3_client = boto3.resource("s3")
sts_client = boto3.client('sts')


client = boto3.client(
    'ses',
    region_name='us-east-2',
)
attachment_string = {}



def checkKey(key,tags):
    if re.search('"Key": "Name"', json.dumps(tags), re.M):
        return 'Y'
    else:
        return 'N'



def lambda_handler(event, context):

 time = datetime.datetime.now().strftime ('%Y-%m-%d-%H-%M-%S')
 s3 = boto3.resource('s3')
 bucket = s3.Bucket('unused-ebs')
 file_name = ('Unused_EBS_Volume_report_' + time + '.csv')
 s3_path = 'EBSReport/' + file_name
 
 fieldnames = ['Tag Name', 'VolumeID', 'VolumeSize', 'VolCreationDate']



 with open('/tmp/file_name', 'w', newline='') as csvFile:
    w = csv.writer(csvFile, dialect='excel')
    w.writerow(fieldnames)
 
    EbsReport1 = []
    isuntagged = False
    EbsReport = "The Following Ebs Volumes are Unused\n For more details, Please download the EBS Volume Report from the s3 Bucket 'unused-ebs'\n\n"
    EbsNoTags = "\n Volume does not have tag - DO NOT DELETE  \n"
    EbsNoTagsReport = "Volume does not have tag - DO NOT DELETE"
    EbsTags = "\n Volume has tag - DO NOT DELETE  \n"
    EbsTagsReport = "Volume has tag - DO NOT DELETE"
    EbsNoNameTags = "\n Volume has No Name as well as DO NOT DELETE tag  \n"
   
    for vol in ec2.volumes.all():
        isuntagged = False
       
        if  vol.state=='available':
           
            # Check for Tags
            #This block will delete the EBS Volumes which has no tags
            #if vol.tags is None:
            #    vid=vol.id
            #    #print ("1 Not Having Tags: {0} for not having Tags".format( vid ))#
            #    volid = vol.id
            #    volsize = vol.size
            #    volcreate = str(vol.create_time.strftime('%Y/%m/%d %H:%M'))
            #                #volcreate  v
            #    EbsReport = str(EbsReport) + str(EbsNoTags) + "- " + str(vol.id) + " - Size: " + str(vol.size) + " - Created: " + str(vol.create_time.strftime('%Y/%m/%d %H:%M'))+ "\n"
            #    raw = [
            #        EbsNoTags,
            #        volid,
            #        volsize,
            #        ]
            #    w.writerow(raw)
            #    raw = []
               
            #        continue
               
           
            #This 'if' block will delete the EBS volumes which has Name tag
            #This 'elseif'  will create Name tag for those EBS volumes which has no Name tag.We are adding the name tag because it was throwing error while deleting EBS volume which has no Name tag
            if checkKey('Name',vol.tags) == 'Y':
                for tag in vol.tags:
                    if  tag['Key'] == globalVars['findNeedle']:
                        value=tag['Value']
                        if value != globalVars['tagsToInclude'] and vol.state == 'available' :

Jude, [3/16/2022 8:45 AM]
EbsReport = str(EbsReport) + str(EbsNoTags) + "- " + str(vol.id) + " - Size: " + str(vol.size) + " - Created: " + str(vol.create_time.strftime('%Y/%m/%d %H:%M'))+ "\n"
                            volid = vol.id
                            volsize = vol.size
                            volcreate = str(vol.create_time.strftime('%Y/%m/%d %H:%M'))
                            raw = [
                                EbsNoTagsReport,
                                volid,
                                volsize,
                                volcreate
                                ]
                            w.writerow(raw)
                            raw = []
                        if value == globalVars['tagsToInclude'] and vol.state == 'available' :
                            volid = vol.id
                            volsize = vol.size
                            volcreate = str(vol.create_time.strftime('%Y/%m/%d %H:%M'))
                            EbsReport = str(EbsReport) + str(EbsTags) + "- " + str(vol.id) + " - Size: " + str(vol.size) + " - Created: " + str(vol.create_time.strftime('%Y/%m/%d %H:%M'))+ "\n"
                            raw = [
                                EbsTagsReport,
                                volid,
                                volsize,
                                volcreate
                                ]
                            w.writerow(raw)
                            raw = []
            else :
                EbsReport = str(EbsReport) + str(EbsNoTags) + "- " + str(vol.id) + " - Size: " + str(vol.size) + " - Created: " + str(vol.create_time.strftime('%Y/%m/%d %H:%M'))+ "\n"
                raw = [
                    EbsNoTagsReport,
                    volid,
                    volsize,
                   volcreate
                    ]
                w.writerow(raw)
                raw = []
                           
 bucket.upload_file('/tmp/file_name', s3_path)
 print(EbsReport)

 


 def publish_sns( EbsReport ):
    print('Publish Messsage to SNS Topic')
    subject_str = 'Alert!Unused EBS Volume Report'
    
    response = snsClient.publish(TargetArn='arn:aws:sns:us-east-2:487696647701:EBS-VOLUME-REPORT',Message=EbsReport,Subject=subject_str,)
    
 #print(EbsReport)
 publish_sns(str(EbsReport))
