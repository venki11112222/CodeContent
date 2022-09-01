
#
# Copyright (c) 2016, Intel Corporation. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
#
# 1. Redistributions of source code must retain the above copyright notice,
# this list of conditions and the following disclaimer.
#
# 2. Redistributions in binary form must reproduce the above copyright notice,
# this list of conditions and the following disclaimer in the documentation
# and/or other materials provided with the distribution. 
#
# 3. Neither the name of the copyright holder nor the names of its contributors
# may be used to endorse or promote products derived from this software without
# specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE
# LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
# CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
# SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
# INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
# CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
# ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE 
# POSSIBILITY OF SUCH DAMAGE.
#

# flag for production or test Database
is_prod = False

# for running at local machine for development & testing
is_localhost = False

# dont change here
# LitePi server machine config
is_lsf_machine = True
is_amr_drive = True
is_smb_drive = False
if is_lsf_machine:
	if not is_amr_drive:
		is_smb_drive = True

# if logging is required
is_logging = False

if not is_localhost:
	from gevent import monkey
	monkey.patch_all()
	from gevent.pywsgi import WSGIServer

	from iamws.token import Token
	from iamws.windows_auth import WindowsAuth
	from iamws.utils import build_endpoint_url

#Flask file for the entire application.
from flask import Flask, Markup, flash
from flask import render_template as render
from flask import request, session, url_for, redirect,make_response
from flask import jsonify,send_file
from werkzeug.utils import secure_filename

import requests
import paramiko
import shutil
from shutil import make_archive

# for excel sheet handling
import openpyxl
from openpyxl.styles import PatternFill, Border, Side, Alignment, Font, Protection
from openpyxl.worksheet.dimensions import ColumnDimension, DimensionHolder
from openpyxl.utils import get_column_letter
from openpyxl.writer.excel import save_virtual_workbook

import MySQLdb
import ssl
import os
import datetime
from isoweek import Week
import json
import smtplib
import email.utils
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from io import BytesIO,StringIO
import time
import io
import csv
import copy
import pytz
import random
import logging
import sys
import traceback
import re
import math

import zipfile
import urllib3
urllib3.disable_warnings()


if is_prod:
	# config file
	file=open("prod_db.txt","r")

	# AMR Storage drive path
	lsf_path = os.path.join("//amr.corp.intel.com","ec","proj","pst","jf","ERAM01","Production","Board","litepi_runs")

else:
	# config file
	file=open("test_db.txt","r")

	# AMR Storage drive path
	lsf_path = os.path.join("//amr.corp.intel.com","ec","proj","pst","jf","ERAM01","Test","Board","litepi_runs")

url_list=file.readlines()
file.close()
API_BASE_URL = url_list[0][0:-1]
HOST_NAME = url_list[1][0:-1]
PORT_NUMBER = int(url_list[2][0:-1])
USER_NAME = url_list[3][0:-1]
PASSWORD = url_list[4][0:-1]
DB_NAME = url_list[5][0:-1]
sys_account = url_list[6][0:-1]
sys_email = url_list[7][0:-1]
sys_pwd = url_list[8][0:-1]
windows_server_hostname = url_list[9][0:-1]
linux_server_hostname = url_list[10][0:-1]
windows_lsf_rds_hostname = url_list[11][0:-1]

# SMB space mount detail
cloud_base_url = url_list[12][0:-1]

print("cloud_base_url: ",cloud_base_url)

# LitePi tool Configs
cadence_license_update_cmd = 'setenv CDS_LIC_FILE "5280@cadence01p.elic.intel.com:5280@cadence02p.elic.intel.com:5280@cadence06p.elic.intel.com"'
cmd_powersi_path = os.path.join("/nfs","site","disks","epsilon","sieve","tools","cadence","sigrity_2021.10","tools.lnx86","bin","powersi")
remote_linux_brd_to_spd_base_path = os.path.join("/nfs","site","disks","ba_mpadsi_work_disk0003","eram","brd_to_spd","")
print("remote_linux_brd_to_spd_base_path: ",remote_linux_brd_to_spd_base_path)

lsf_path_smb = ""

'''
if is_lsf_machine:
	if is_amr_drive:
		#lsf_path = os.path.join("//amr.corp.intel.com","ec","proj","pst","jf","ERAM01","litepi_runs")
		lsf_path = os.path.join("//amr.corp.intel.com","ec","proj","pst","jf","ERAM01","litepi_runs")
	else:
		lsf_path = os.path.join("//eramspace-DM.cps.intel.com","datashare5","eram","litepi")
		lsf_path_smb = os.path.join(cloud_base_url,"eram","litepi")
else:
	lsf_path = os.path.join("C:","Eram","LitePi")
'''
#lite_pi_path = os.path.join("C:","tools","LitePI","TestRelease_2_13","Lite-PI-Cmd.exe")

compare_parallel_error = "Cannot access 'Book1.xlsx'"
extract_im_error = "Looks like output files are not generated by ExtractIM."
no_license_available = "No license available."


windows_lsf_rds = "windows_lsf_rds"
linux = "linux"

REQUEST_ID=0
app = Flask(__name__)
email_id=""

tz = pytz.timezone('Asia/Kolkata')

firstHeader = 'FFD966'
secondHeader = 'DBDBDB'
horizontal = "center"
vertical = "center"

if is_logging:

	log_file_path = os.path.join(cloud_base_url,"logs","eram_board_app.log")

	# log file config part
	logging.basicConfig(filename=log_file_path,
	                    format='%(asctime)s - %(levelname)s - %(message)s',
	                    datefmt='[%d-%b-%y %H:%M:%S - %a]',
	                    level=logging.INFO)

# log uncaught exceptions
def log_exceptions(type, value, tb):

	msg = '-' * 75
	msg += '\n'
	for line in traceback.TracebackException(type, value, tb).format(chain=True):
		msg += str(line)
	logging.exception(msg)

	sys.__excepthook__(type, value, tb) # calls default excepthook

if is_logging:
	# calls log function for any uncaught error
	sys.excepthook = log_exceptions

db = MySQLdb.connect(host=HOST_NAME,  # your host
					user=USER_NAME,  # username
					passwd=PASSWORD, # password
					db=DB_NAME,		 # db name
					port=PORT_NUMBER,#port no.
					ssl={'ssl':
					{'ca':'Certificates/cert.pem',
					'key':'Certificates/cert.key'}},
					use_unicode=True, 
					charset="utf8")

mail_server = smtplib.SMTP('smtpauth.intel.com',587)
mail_server.ehlo()
mail_server.starttls()
mail_server.ehlo()
mail_server.login(sys_email,sys_pwd)

global inactive_email_ids

inactive_email_ids = []

def set_inactive_emailids():

	global inactive_email_ids

	temp = []
	sql="SELECT DISTINCT a.EmailID FROM HomeTable a WHERE a.IsActive = %s"
	val = (0,)
	active_users=execute_query(sql,val)

	for i in range(0,len(active_users)):
		temp.append(active_users[i][0])

	inactive_email_ids = copy.deepcopy(temp)

	return True

def send_mail(reciever,subject,message,email_list=[]):

	global inactive_email_ids

	# avoid inactive users
	if reciever in inactive_email_ids:
		return False

	mail_recipients_list = ''
	if reciever in ['shivani.r.jain@intel.com','vadivelx.balakrishnan@intel.com','madhumithax.karado@intel.com']:

		for i in email_list:
			if i not in inactive_email_ids:
				mail_recipients_list += '<br>'+i

		message +=  '<br><br><b><u>Mail Recipients:</u></b><br>'+mail_recipients_list

	if is_prod:
		mail_subject = 'BRD - '+subject

	else:
		mail_subject = 'BRD TEST - ' + subject
		reciever = ','.join(['vadivelx.balakrishnan@intel.com','madhumithax.karado@intel.com'])

	sender=sys_email
	sender_name="eram"

	msg = MIMEMultipart('alternative')
	msg['Subject'] = mail_subject
	#msg['From'] = sender
	msg['From'] = email.utils.formataddr((sender_name, sender))
	msg['To'] = reciever
	if(reciever != "na@intel.com"):
		
		html = """\

		<html>

		<body>

		<div><span style="font-size: 14px;font-weight: normal;font-family: Calibri, sans-serif;">"""+ message +""" </span></div>
		<div style="color: gray;"><span style="font-size: 13px;font-weight: normal;font-family: Calibri, sans-serif;"><br>This is an auto generated email. This mailbox is not monitored.<br>For any queries, please <a href="mailto:shivani.r.jain@intel.com;vadivelx.balakrishnan@intel.com;">click here</a> or contact shivani.r.jain@intel.com, vadivelx.balakrishnan@intel.com</span><br><br></div>

		</html>

		"""		
		part1 = MIMEText(html, 'html')

		msg.attach(part1)

		global mail_server

		text = msg.as_string()
		try:
			mail_server.sendmail(sender, reciever.split(','), text)
			print("sending mail...")
		except Exception as inst:
			mail_server = smtplib.SMTP('smtpauth.intel.com',587)
			mail_server.ehlo()
			mail_server.starttls()
			mail_server.ehlo()
			mail_server.login(sys_email,sys_pwd)
			#text = msg.as_string()
			print("sending mail...")
			mail_server.sendmail(sender, reciever.split(','), text)

		#server.quit()

def send_mail_html(reciever,subject,message,email_list=[]):

	global inactive_email_ids

	# avoid inactive users
	if reciever in inactive_email_ids:
		return False

	mail_recipients_list = ''
	if reciever in ['shivani.r.jain@intel.com','vadivelx.balakrishnan@intel.com','madhumithax.karado@intel.com']:

		for i in email_list:
			if i not in inactive_email_ids:
				mail_recipients_list += '<br>'+i

		message +=  '<br><br><b><u>Mail Recipients:</u></b><br>'+mail_recipients_list

	if is_prod:
		mail_subject = 'BRD - '+subject

	else:
		mail_subject = 'BRD TEST - ' + subject
		reciever = ','.join(['vadivelx.balakrishnan@intel.com','madhumithax.karado@intel.com'])

	sender=sys_email
	sender_name="eram"

	msg = MIMEMultipart('alternative')
	msg['Subject'] = mail_subject
	msg['From'] = email.utils.formataddr((sender_name, sender))
	msg['To'] = reciever
	if(reciever != "na@intel.com"):
		
		text = ''
		html = """\

		<html>

		<body>

		<div><span style="font-size: 14px;font-weight: normal;font-family: Calibri, sans-serif;">"""+ message +""" </span></div>
		<div style="color: gray;"><span style="font-size: 13px;font-weight: normal;font-family: Calibri, sans-serif;"><br>This is an auto generated email. This mailbox is not monitored.<br>For any queries, please <a href="mailto:shivani.r.jain@intel.com;vadivelx.balakrishnan@intel.com;">click here</a> or contact shivani.r.jain@intel.com, vadivelx.balakrishnan@intel.com</span><br><br></div>

		</html>

		"""
		
		part1 = MIMEText(text, 'plain')
		part2 = MIMEText(html, 'html')

		msg.attach(part1)
		msg.attach(part2)

		global mail_server

		text = msg.as_string()
		try:
			mail_server.sendmail(sender, reciever.split(','), text)
			print("sending html mail...")
		except Exception as inst:
			mail_server = smtplib.SMTP('smtpauth.intel.com',587)
			mail_server.ehlo()
			mail_server.starttls()
			mail_server.ehlo()
			mail_server.login(sys_email,sys_pwd)
			#text = msg.as_string()
			print("sending html mail...")
			mail_server.sendmail(sender, reciever.split(','), text)

def get_server_machine_connection(server="windows"):

	if server == "windows":
		server_hostname = copy.deepcopy(windows_server_hostname)

	elif server == "linux":
		server_hostname = copy.deepcopy(linux_server_hostname)

	elif server == "windows_lsf_rds":
		server_hostname = copy.deepcopy(windows_lsf_rds_hostname)

	else:
		print("Incorrect Server details passed.")
		return False

	try:
		ssh = paramiko.SSHClient()
		ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
		ssh.connect(server_hostname,username=sys_account,password=sys_pwd,banner_timeout=200)
		#print("Connected to %s" % server_hostname)
		return ssh

	except paramiko.AuthenticationException:
		print("Failed to connect to %s due to wrong username/password" %server_hostname)
		return False

	except Exception as inst:
		print(inst)
		if is_logging:
			logging.exception('')
		return False

	return False

ssh_file_transfer = None
sftp_file_transfer = None

def push_file_to_windows_server(boardid=0,rev="rev",comp_name="",local_file_path="",remote_file_path="",lsf_path="",loop_count=0):

	global ssh_file_transfer
	global sftp_file_transfer

	try:

		sftp_file_transfer.put(local_file_path,remote_file_path)

	except IOError:

		# create directory if not exists
		try:
			remote_file_dir = os.path.join(lsf_path,"Board_ID_"+str(boardid))
			sftp_file_transfer.mkdir(remote_file_dir)
		except:
			pass

		try:
			remote_file_dir = os.path.join(lsf_path,"Board_ID_"+str(boardid),rev)
			sftp_file_transfer.mkdir(remote_file_dir)
		except:
			pass

		try:
			remote_file_dir = os.path.join(lsf_path,"Board_ID_"+str(boardid),rev,comp_name)
			sftp_file_transfer.mkdir(remote_file_dir)
		except:
			pass

		# after creating directory, try pushing the file
		sftp_file_transfer.put(local_file_path,remote_file_path)

	except Exception as inst:

		# create ssh connection
		if is_lsf_machine:
			ssh_file_transfer = get_server_machine_connection(server="windows_lsf_rds")
		else:
			ssh_file_transfer = get_server_machine_connection(server="windows")

		sftp_file_transfer = ssh_file_transfer.open_sftp()

		loop_count += 1

		if loop_count < 3:
			print("connecting again for sftp client..")
			push_file_to_windows_server(boardid=boardid,rev=rev,comp_name=comp_name,local_file_path=local_file_path,remote_file_path=remote_file_path,lsf_path=lsf_path,loop_count=loop_count)

		else:
			if is_logging:
				logging.exception('')
			print(inst)

	return True

def push_file_to_linux_server(boardid=0,local_file_path="",remote_file_path="",remote_dir=""):

	# create ssh connection
	ssh = get_server_machine_connection(server="linux")

	try:
		ftp_client = ssh.open_sftp()
		ftp_client.put(local_file_path,remote_file_path)

	except IOError:

		# create directory if not exists
		try:
			remote_file_dir = copy.deepcopy(remote_dir)
			ftp_client.mkdir(remote_file_dir)
		except Exception as inst:
			print(inst)
			return False

		try:
			# after creating directory, try pushing the file
			ftp_client.put(local_file_path,remote_file_path)
		
		except Exception as inst:
			print(inst)
			return False

	except Exception as inst:
		print(inst)
		return False

	# terminate ssh connection
	try:
		ftp_client.close()
		close_ssh_connection(ssh=ssh)
	except Exception as inst:
		print(inst)

	return True

ssh_rds_machine = None

def exec_command_through_ssh_trigger_job(server="windows",cmd="",need_response=True,close_ssh=False):

	global ssh_rds_machine

	try:
		print("Executing command on remote workstation: ",cmd)
		stdin, stdout, stderr = ssh_rds_machine.exec_command(cmd,get_pty=True)

	except Exception as inst:

		if is_logging:
			logging.exception('')
		print(inst)

		server_hostname = copy.deepcopy(windows_lsf_rds_hostname)

		try:
			print("trying again to connect...")
			ssh_rds_machine = paramiko.SSHClient()
			ssh_rds_machine.set_missing_host_key_policy(paramiko.AutoAddPolicy())
			ssh_rds_machine.connect(server_hostname,username=sys_account,password=sys_pwd)
			print("Connected for trigger job to %s" % server_hostname)

			print("Executing command on remote workstation: ",cmd)
			stdin, stdout, stderr = ssh_rds_machine.exec_command(cmd,get_pty=True)

		except Exception as inst:
			print(inst)

	if need_response:
		print("reading responses...")
		try:
			err = ''.join(stderr.readlines())
			out = ''.join(stdout.readlines())
			final_output = str(out)+str(err)
			print(final_output)
		except Exception as inst:
			if is_logging:
				logging.exception('')
			print(inst)

	# terminate ssh connection
	if close_ssh:
		close_ssh_connection(ssh=ssh_rds_machine)

	return True

def exec_command_through_ssh(server="windows",cmd="",need_response=True,close_ssh=True):

	# create ssh connection
	ssh = get_server_machine_connection(server=server)

	try:
		print("Executing command on remote workstation: ",cmd)
		stdin, stdout, stderr = ssh.exec_command(cmd,get_pty=True)
	except Exception as inst:
		if is_logging:
			logging.exception('')
		print(inst)
		return False

	if need_response:
		print("reading responses...")
		try:
			err = ''.join(stderr.readlines())
			out = ''.join(stdout.readlines())
			final_output = str(out)+str(err)
			print(final_output)
		except Exception as inst:
			if is_logging:
				logging.exception('')
			print(inst)

	# terminate ssh connection
	if close_ssh:
		close_ssh_connection(ssh=ssh)

	return True

def close_ssh_connection(ssh):
	# close the connection
	try:
		ssh.close()
		#print("OpenSSH connection closed.")

	except Exception as inst:
		if is_logging:
			logging.exception('')
		print(inst)

def execute_query_many(sql,value):
	try:
		global db
		cur = db.cursor()
		cur.executemany(sql,value)
		db.commit()
		#result = cur.fetchall()
		print("Rows affected: ",cur.rowcount)
		#print("result: ",result)
		return cur.rowcount

	except(AttributeError, MySQLdb.OperationalError):

		db = MySQLdb.connect(host=HOST_NAME,  # your host 
                    user=USER_NAME,  # username
                    passwd=PASSWORD, # password
                    db=DB_NAME,		 # db name	
                    port=PORT_NUMBER,#port no.
                    ssl={'ssl':
                    {'ca':'Certificates/cert.pem',
                    'key':'Certificates/cert.key'}},
                    use_unicode=True, 
					charset="utf8")

		cur = db.cursor()
		cur.executemany(sql,value)
		db.commit()
		#result = cur.fetchall()
		print("Rows affected: ",cur.rowcount)
		#print("result: ",result)
		return cur.rowcount

def execute_query(sql,value):
	try:
		global db
		cur = db.cursor()
		cur.execute(sql,value)
		db.commit()
		result = cur.fetchall()
		return result

	except(AttributeError, MySQLdb.OperationalError):

		db = MySQLdb.connect(host=HOST_NAME,  # your host 
                    user=USER_NAME,  # username
                    passwd=PASSWORD, # password
                    db=DB_NAME,		 # db name	
                    port=PORT_NUMBER,#port no.
                    ssl={'ssl':
                    {'ca':'Certificates/cert.pem',
                    'key':'Certificates/cert.key'}},
                    use_unicode=True, 
					charset="utf8")

		cur = db.cursor()
		cur.execute(sql,value)
		db.commit()
		result = cur.fetchall()
		return result

def execute_query_sql(sql):
	try:
		global db
		cur = db.cursor()
		cur.execute(sql)
		db.commit()
		result = cur.fetchall()
		return result

	except(AttributeError, MySQLdb.OperationalError):

		db = MySQLdb.connect(host=HOST_NAME,  # your host 
                    user=USER_NAME,  # username
                    passwd=PASSWORD, # password
                    db=DB_NAME,		 # db name	
                    port=PORT_NUMBER,#port no.
                    ssl={'ssl':
                    {'ca':'Certificates/cert.pem',
                    'key':'Certificates/cert.key'}},
                     use_unicode=True, 
					charset="utf8")

		cur = db.cursor()
		cur.execute(sql)
		db.commit()
		result = cur.fetchall()
		return result

def valid_date(datestring):
	try:
		datetime.datetime.strptime(datestring, '%Y-%m-%d')
		return True
	except ValueError:
		if is_logging:
			logging.exception('')
		return False

#Method called when the home screen is loaded.
@app.route('/',methods = ['POST', 'GET'])
def main_home_page():

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	if is_prod:
		return render("home_page_prod.html",region_name=region_name)
	else:
		return render("home_page.html",region_name=region_name)

@app.route("/board_home",methods = ['POST', 'GET'])
def index():

	#session['wwid'] = None
	#session['is_admin'] = None

	if is_prod:
		region_name = ""
	else:
		region_name = copy.deepcopy(DB_NAME)

	# automation - updating design status to 'Projected and No updates' for below conditions,
	# a.	If design files are not uploaded and design status is ERAM timeline commit or Design Team Projection
	# b.	Design start date < current date - change the design status to 'Projected and No updates' (trigger email and use the same email/recipient as of update design timelines)
	check_design_status()


	# automation - On design files upload for yet to kickstart design - one time activity
	#i.	Projected & No updates : change the start date as current date and update end date accordingly. No email trigger.  Change status to Design Team Projection
	updte_design_status_and_dates()

	# reminder mail automation:
	#Automate reminder email for ongoing designs - sent only once
	#	a.	Send reminder email on end-date 8:00 AM IST
	reminder_automation()


	# Design is in yet to kickstart state and current date = start date - 2days : Package WIP 
	#	a.	Reminder to design team on upcoming design to upload the files or update the design timelines
	#	b.	Trigger multiple reminder emails until design is in 'Design Team Projection' and 'ERAM Timeline commit' and last request update >= 3 days before review start date.

	reminder_automation_upload_files()

	sql = "SELECT BoardID FROM DesignCalendar WHERE BoardState = 'Design Review In-Progress' AND ProposedEndDate < (SELECT DATE(DATE_ADD(NOW(),INTERVAL 750 MINUTE)))"
	in_progress_designs = execute_query_sql(sql)

	if(in_progress_designs!=()):
		for i in in_progress_designs:

			sql = "UPDATE DesignCalendar SET ProposedEndDate = (SELECT DATE(DATE_ADD(NOW(),INTERVAL 750 MINUTE))) WHERE BoardState = 'Design Review In-Progress' AND BoardID = %s"
			val = (i,)
			in_progr_result = execute_query(sql,val)

	start_calender_date = request.form.get("start_calender_date")
	end_calender_date = request.form.get("end_calender_date")

	start_date_default = start_calender_date
	end_date_default = end_calender_date

	sql = "SELECT MIN(ProposedStartDate) FROM DesignCalendar"
	start_date_min = execute_query_sql(sql)[0][0]

	sql = "SELECT MAX(ProposedEndDate) FROM DesignCalendar"
	start_date_max = execute_query_sql(sql)[0][0]

	end_date_min = start_date_min
	end_date_max = start_date_max

	start_date_default_ww = ''
	end_date_default_ww = ''

	max_work_week_display = 18
	#cal_table_width = 3000	# css style for table width

	filter_option_enabled = False

	# date filter option
	if (start_calender_date == None or end_calender_date == None):
		#sql = "SELECT MIN(ProposedStartDate) FROM DesignCalendar D, ScheduleTable S WHERE ScheduleStatusID <> 1 AND D.BoardID = S.BoardID"
		sql = "SELECT MIN(ProposedStartDate) FROM DesignCalendar WHERE BoardState NOT IN ('Design Signed-off','Design Not signed-off')"
		try:
			m1 = execute_query_sql(sql)[0][0]
		except:
			m1 = datetime.datetime.now(tz).date()

		if m1 is None:
			m1 = datetime.datetime.now(tz).date()

		print("m1 before: ",m1)

		# if ongoing design is there, but its starting date is not comes before 3 weeks timeframe, then we can re-adjust the start date of calender, so that it can display from 3 weeks before from today
		if m1 > datetime.datetime.now(tz).date() - datetime.timedelta(days=14):
			m1 = datetime.datetime.now(tz).date() - datetime.timedelta(days=14)

		print("m1111111: ",m1)

		# if no ongoing design, then we should start calender view with 3 weeks before from current week.
		sql = "SELECT * FROM DesignCalendar WHERE BoardState = %s"
		val = ("Design Review In-Progress",)
		check_for_ongoing_design = execute_query(sql,val)
		print("check_for_ongoing_design: ",check_for_ongoing_design)

		if check_for_ongoing_design == ():
			m1 = datetime.datetime.now(tz).date() - datetime.timedelta(days=14)

		print("m1: ",m1)
		m1_ww = float(get_work_week_fun(m1)) - 2
		if(m1_ww < 1):
			m1_ww = 1

		print("m1_ww:",m1_ww)
		year = m1.year
		m2 = get_date_from_work_week_fun(year=year,ww=m1_ww)
	else:
		filter_option_enabled = True
		m2 = datetime.datetime.strptime(start_calender_date, '%Y-%m-%d')
		year = m2.year

		start_date_default_ww = str(get_work_week_fun_with_year(m2))

	print("m2   : ",m2)
	print("year: ",year)

	print("m2:",m2)
	# to get date from Monday on that week
	#m2_ww = int(float(get_work_week_fun(m2)))
	m2_ww = int(float(get_work_week_fun(m2)))-1
	if m2_ww < 1:
		m2_ww = 1

	print("m2_ww:",m2_ww)

	m3 = get_date_from_work_week_fun(year=year,ww=m2_ww)
	print("m3:",m3)

	# date filter option
	if filter_option_enabled:
		end_calender_date_format = datetime.datetime.strptime(end_calender_date, '%Y-%m-%d')
		end_date_default_ww = str(get_work_week_fun_with_year(end_calender_date_format))
		m4_ww = int(float(get_work_week_fun(end_calender_date_format)))
		max_end_date = copy.deepcopy(end_calender_date_format.date())
		#print("m4_ww: ",m4_ww)
		diff_temp = 0
		#print("get_isocalendar(max_end_date)[2]: ",get_isocalendar(max_end_date)[2])
		if get_isocalendar(max_end_date)[2] == 7:
			diff_temp = -1
		else:
			while get_isocalendar(max_end_date)[2] != 7:
				max_end_date += datetime.timedelta(days=1)
				#print("max_end_date-------: ",max_end_date)

		#ww_for_max_end = m4_ww+0.6 # to get saturday as last end day
		#ww_for_max_end = m4_ww+1 # to get saturday as last end day
		#max_end_date = get_date_from_work_week_fun(year=end_calender_date_format.year,ww=ww_for_max_end)
		#print("end_calender_date_format: ",end_calender_date_format)
		#print("max_end_date: ",max_end_date)

		#print("m4_ww: ",m4_ww)
		#print("m2_ww: ",m2_ww)
		# to decide number of weeks to be displayed
		#max_work_week_display = m4_ww - m2_ww + 1
		max_work_week_display = m4_ww - m2_ww

		# to check for negative number if cross between years
		if max_work_week_display < 0:
			days_diff_temp = m3 - end_calender_date_format.date()
			#print("days_diff_temp: ",days_diff_temp.days)
			days_diff_temp_ww = 0 - int(days_diff_temp.days/7)
			max_work_week_display = days_diff_temp_ww + 1

		#if(max_work_week_display > 24):
		#	max_work_week_display = 24
		if(max_work_week_display > 19):
			max_work_week_display = 19

		# set max limit for end date value in filter
		end_date_min = start_calender_date
		end_date_max = datetime.datetime.strptime(start_calender_date, '%Y-%m-%d').date() + datetime.timedelta(days=120)
	else:
		max_end_date = m3 + datetime.timedelta(days=126)

	#print("max_work_week_display: ",max_work_week_display)

	#if (max_work_week_display != 18):
	#cal_table_width = 600 + int(max_work_week_display * 140)
	cal_table_width = 730 + int(max_work_week_display * 140)

	#sql="SELECT a.BoardStateColor,a.BoardStateName,b.ProposedStartDate,b.ProposedEndDate,c.BoardName,b.BoardID from BoardStateCalendar a,DesignCalendar b,BoardDetails c where a.BoardStateName=b.BoardState and b.BoardID = c.BoardID and b.ProposedStartDate >= %s order by b.BoardID asc"
	#sql="SELECT a.BoardStateColor,a.BoardStateName,b.ProposedStartDate,b.ProposedEndDate,c.BoardName,c.BoardID from BoardStateCalendar a,DesignCalendar b,BoardDetails c where a.BoardStateName=b.BoardState and b.BoardID = c.BoardID and b.ProposedEndDate >= %s and b.ProposedStartDate <= %s ORDER BY b.ProposedStartDate ASC,c.BoardID ASC"
	sql="SELECT a.BoardStateColor,a.BoardStateName,b.ProposedStartDate,b.ProposedEndDate,c.BoardName,c.BoardID from BoardStateCalendar a,DesignCalendar b,BoardDetails c where a.BoardStateName=b.BoardState and b.BoardID = c.BoardID and b.ProposedEndDate >= %s and b.ProposedStartDate <= %s ORDER BY FIELD(c.DesignTypeID,3,4,5,21,1,2,6,7,8,17,18,19,20,22,23,24,25,26,27,28,29,30,31,32,33),c.PlatformID ASC,c.BoardID ASC"
	val = (m3,max_end_date)
	print("val index:",val)
	color = execute_query(sql,val)
	color_list=[]
	status_hover = []
	design_id_list = []
	for i in range(len(color)):
		col = color[i][0]
		color_list.append(col)
		stat = 'Click to view design summary\n\n'
		stat += 'Status: ' + color[i][1] + '\nTimelines: '
		try:
			stat += get_work_week_fun_with_year(color[i][2]) + ' - ' + get_work_week_fun_with_year(color[i][3])
		except:
			pass
		
		status_hover.append(stat)
		design_id_list.append(str(color[i][5]))

	#sql = "SELECT ProposedStartDate,ProposedEndDate,BoardID FROM DesignCalendar D WHERE D.ProposedEndDate >= %s AND D.ProposedStartDate <= %s ORDER BY D.ProposedStartDate ASC,D.BoardID ASC"
	sql = "SELECT D.ProposedStartDate,D.ProposedEndDate,D.BoardID FROM DesignCalendar D,BoardDetails a WHERE a.BoardID = D.BoardID AND D.ProposedEndDate >= %s AND D.ProposedStartDate <= %s ORDER BY FIELD(a.DesignTypeID,3,4,5,21,1,2,6,7,8,17,18,19,20,22,23,24,25,26,27,28,29,30,31,32,33),a.PlatformID ASC,a.BoardID ASC"
	val = (m3,max_end_date)
	dates = execute_query(sql,val)		
			
	start_date_list = []
	end_date_list=[]
	startday = []
	endday=[]
	endyear=[]
	startyear=[]
	for_minstart=[]
	minstartweek = 0
	minstartyear = 0
	name_board=[]
	startweek=[]
	endweek=[]
	dayscount=[]

	if(dates != ()):
		for i in range(len(dates)):

			if(dates[i][0] < m3):
				sdat = m3
			else:
				sdat = dates[i][0]

			if(dates[i][1] > max_end_date):
				edat = max_end_date
			else:
				edat=dates[i][1]

			start_date_list.append(float(str(get_isocalendar(sdat)[1])+'.'+str(get_isocalendar(sdat)[2])))   ####getting workweek from date in db. format is pretty different hence the typecasting
			end_date_list.append(float(str(get_isocalendar(edat)[1])+'.'+str(get_isocalendar(edat)[2])))
			startday.append(int(str(get_isocalendar(sdat)[2])))
			endday.append(int(str(get_isocalendar(edat)[2])))
			endyear.append(int(str(get_isocalendar(edat)[0])))
			startyear.append(int(str(get_isocalendar(sdat)[0])))
			for_minstart.append(sdat)

		minstart=min(for_minstart)
		minstartweek=get_isocalendar(minstart)[1]
		minstartyear=int(str(get_isocalendar(minstart)[0])[2:4])

		wwstart=int(min(start_date_list))           ##wwstart = min of the pending boards starting date

		#print("wwstart: ",wwstart)
		#print("m2_ww: ",m2_ww)
		if filter_option_enabled:
			diff = int(wwstart - m2_ww)
			#print("diff: ",diff)
			#print("max_work_week_display : ",max_work_week_display)
			#if diff > 0:
			#	if max_work_week_display > diff:
			#		max_work_week_display -= diff
								
		#print("max_work_week_display::::",max_work_week_display)	
		'''
		sql="SELECT a.BoardName FROM BoardDetails a, DesignCalendar b WHERE a.BoardID=b.BoardID and b.ProposedEndDate >= %s and b.ProposedStartDate <= %s ORDER BY FIELD(b.BoardState,'Projected & No-Updates','Design Review In-Progress','Design Not signed-off','Design Signed-off','ERAM Timeline Commit','Design Team Projection'),b.ProposedStartDate ASC"
		val = (m3,max_end_date)
		board_name = execute_query(sql,val)
		name_board=[]
		for i in board_name:
			name_board.append(i[0])
		'''

		#sql="SELECT a.BoardName,c.PlatformName,d.DesignTypeName,a.BoardID FROM BoardDetails a, DesignCalendar b,Platform c,DesignType d WHERE a.BoardID=b.BoardID and a.PlatformID=c.PlatformID and a.DesignTypeID=d.DesignTypeID AND b.ProposedEndDate >= %s AND b.ProposedStartDate <= %s ORDER BY b.ProposedStartDate ASC,a.BoardID ASC"
		sql="SELECT a.BoardName,c.PlatformName,d.DesignTypeName,a.BoardID FROM BoardDetails a, DesignCalendar b,Platform c,DesignType d WHERE a.BoardID=b.BoardID and a.PlatformID=c.PlatformID and a.DesignTypeID=d.DesignTypeID AND b.ProposedEndDate >= %s AND b.ProposedStartDate <= %s ORDER BY FIELD(a.DesignTypeID,3,4,5,21,1,2,6,7,8,17,18,19,20,22,23,24,25,26,27,28,29,30,31,32,33),a.PlatformID ASC,a.BoardID ASC"
		val = (m3,max_end_date)
		board_cred=execute_query(sql,val)


		startweek=[]
		endweek=[]
		dayscount=[]
		for i in start_date_list:
			startweek.append(int(i))

		for i in end_date_list:
			endweek.append(int(i))

		for i in range(len(startweek)):
			if (endyear[i] > startyear[i]):
				endweek[i] = endweek[i] + 52

			if (endweek[i] > startweek[i]):
				days = (endweek[i] - startweek[i] - 1) * 5 + endday[i] + ( 5 -startday[i] +1 )
			elif ( endweek[i] == startweek[i]):
				days = endday[i] - startday[i]+ 1
			else:
				print("error in date. Start week:" + str(startweek[i]) + "End Week:" + str(endweek[i]) + "Design ID:" + str(dates[i][2]))
				days = 0
			dayscount.append(days)
	else:
		board_cred = ()


	name = session.get('username')
	query="SELECT bd.BoardName,bd.BoardID from BoardDetails bd,ScheduleTable st where st.BoardID=bd.BoardID and st.ScheduleStatusID in (2,3,4,6) order by bd.BoardID asc"
	ids = execute_query_sql(query)

	id_list = []
	board_id_list = []
	for i in range(len(ids)):
		idd = '[ID: '+str(ids[i][1])+'] - '+str(ids[i][0])
		id_list.append(idd)
		board_id_list.append(ids[i][1])

	query = "SELECT BoardStateName FROM BoardStateCalendar"
	statename = execute_query_sql(query)

	statename_list = []
	for i in range(len(statename)):
		state = statename[i][0]
		statename_list.append(state)

	if session.get('wwid'):

		wwid = session.get('wwid')

		username = session.get('username')
		user_role_name = session.get('user_role_name')
		region_name = session.get('region_name')
		error_msg_inactive_user = session.get('inactive_user_msg')

		# to check for user status - active or inactive
		if session.get('is_inactive_user'):
			return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

		sql = "SELECT * FROM RequestAccess WHERE WWID = %s AND StateID IN  ( SELECT StateID FROM RequestState WHERE StateName= %s)"
		values = (wwid,'Accepted')
		accepted = execute_query(sql,values)
		if(accepted):
			sql = "select AdminAccess from HomeTable where wwid=%s"
			val = (wwid,)
			access = execute_query(sql, val)
			if (access[0][0] == "yes"):
				visible = "initial"
			else:
				visible = "none"

			sql="SELECT a.RoleID,b.RoleName FROM HomeTable a LEFT JOIN RoleTable b ON a.RoleID=b.RoleID WHERE WWID=%s "
			val = (wwid,)
			role_id=execute_query(sql,val)

			if (role_id[0][0] == 14):
				mgt_access=True
			else:
				mgt_access=False

			role_name = role_id[0][1]

			today_date_ww = str(get_work_week_fun(datetime.datetime.now(tz).date())).split('.')
			ww_number = int(today_date_ww[0])
			ww_day_number = int(today_date_ww[1])

			username = session.get('username')
			user_role_name = session.get('user_role_name')
			region_name = session.get('region_name')

			#print("max_work_week_display: ",max_work_week_display)
			#print("startww: ",int(minstartweek))
			#print("minstartyear: ",minstartyear)
			#print("ww_number: ",ww_number)
			return render('index.html',minstartyear=minstartyear,username=username,user_role_name=user_role_name,design_id_list=design_id_list,ww_number=ww_number,ww_day_number=ww_day_number,visible=visible,region_name=region_name,start_date_min=start_date_min,start_date_max=start_date_max,end_date_min=end_date_min,end_date_max=end_date_max,cal_table_width=cal_table_width,start_date_default=start_date_default,end_date_default=end_date_default,start_date_default_ww=start_date_default_ww,end_date_default_ww=end_date_default_ww,max_work_week_display=max_work_week_display,board_cred=board_cred,mgt_access=mgt_access,name=name,id_list=id_list,board_id_list=board_id_list,statename_list=statename_list,startww=int(minstartweek),startday=startday,startweek=startweek,dayscount=dayscount,color_list=color_list,status_hover=status_hover)
		
		values = (wwid,'Pending')
		pending = execute_query(sql,values)
		if(pending):
			sql = "SELECT RequestID FROM RequestAccess WHERE WWID = %s"
			val = (wwid,)
			requestid = execute_query(sql,val)
			username = session.get('username')
			user_role_name = session.get('user_role_name')
			region_name = session.get('region_name')
			return render('pending.html',requestid=requestid[0][0],username=username,user_role_name=user_role_name,region_name=region_name)

		values = (wwid,'Rejected')
		rejected = execute_query(sql,values)
		if(rejected):
			return render('rejected.html')

		sql = "SELECT * FROM RequestAccess WHERE WWID = %s"
		values = (wwid,)
		rs_temp = execute_query(sql,values)
		if rs_temp == () and wwid is not None:
			sql = "SELECT RoleName from RoleTable WHERE RoleID <> 2 AND RoleID <> 11 "
			roles = execute_query_sql(sql)
			roles_list=[]
			for i in roles:
				role=i[0]
				roles_list.append(role)
			return render('request_access.html',role=roles_list,message="")
	else:

		if is_localhost:
			return redirect(url_for('login', _external=True))
		else:
			session['target_page'] = 'index'
			sso_url = url_for('sso', _external=True)
			windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
			redirect_url = windows_auth_url + '?redirecturl=' + sso_url
			return redirect(redirect_url)

@app.route("/reminder",methods = ['POST', 'GET'])
def reminder():
	boardid = request.form.get("boardid")
	subject = request.form.get("subject")
	message = request.form.get("message")
	usernames = request.form.getlist("usernames")

	wwid = session.get('wwid')

	# log table
	try:
		log_notes = 'User has triggered reminder mail for Design ID: '+str(boardid)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
		val = ('Reminder',boardid,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")

	sql = "SELECT ScheduleStatusID FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	try:
		state = execute_query(sql,val)[0][0]
	except:
		state = 2

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	try:
		boardname = execute_query(sql,val)[0][0]
	except:
		boardname = ''

	email_list = []
	user_specific_comp_name = []

	eids = []
	if(usernames):
		sql = "select EmailID from HomeTable where WWID in %s"
		val = (usernames,)
		eids = execute_query(sql, val)

	for i in range(len(eids)):
		if(eids[i][0]):
			email_list.append(eids[i][0])	

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
	val = (boardid,)
	designlist = execute_query(query,val)
	designlead_list = []
	for i in range(len(designlist)):
		eid = designlist[0][1]
		email_list.append(eid)

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.Username=b.DesignManagerWWID and b.BoardID AND b.BoardID = %s) LIMIT 1"
	val = (boardid,)
	designmanager = execute_query(query,val)
	if designmanager != ():
		email_list.append(designmanager[0][1])

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
	val = (boardid,)
	cadlist = execute_query(query,val)
	cadlead_list = []
	for i in range(len(cadlist)):
		eid = cadlist[0][1]
		email_list.append(eid)


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.PIFLeadWWID and b.BoardID AND b.BoardID = %s)"
	piflist = execute_query(query,val)
	piflead_list = []
	for i in range(len(piflist)):
		eid = piflist[0][1]
		email_list.append(eid)

	# all pif leads
	email_list += get_pif_leads_email_id_by_board_id(boardid=boardid)

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val=('yes',)
	admin_list = execute_query(sql,val)				

	for j in admin_list:
		email_list.append(j[0])


	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)

	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	rid = execute_query(sql,val)[0][0]

	timeline = ""
	if(rid == 1):
		timeline = "Core/Placement layout review Rev0p6"
	else:
		timeline = "Full layout review Rev1p0"

	sql="SELECT a.CategoryName,b.EmailID,C3.CategoryLeadWWID1 from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2,BoardReviewDesigner B1,ScheduleTableComponent S1 WHERE cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID AND C2.ComponentID = B1.ComponentID AND B1.BoardID = S1.BoardID AND B1.ComponentID = S1.ComponentID AND B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND S1.ScheduleStatusID IN (2,3,6,NULL) ORDER BY cr.ComponentID"
	val=(sku_plat[0][0], sku_plat[0][1],sku_plat[0][2], sku_plat[0][3],sku_plat[0][0], sku_plat[0][1],sku_plat[0][2], sku_plat[0][3],boardid,"yes")
	catlead=execute_query(sql,val)
	if(catlead != ()):
		for i in catlead:
			email_list.append(i[1])		

			if i[2] is not None:
				if i[2] != []:
					cat_sec_wwid_list = i[2][1:-1].split(', ')

					for j in range(0,len(cat_sec_wwid_list)):
						like_user_wwid = '%' + str(cat_sec_wwid_list[j]) + '%'
						if cat_sec_wwid_list[j] not in ['99999999',99999999]:
							sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
							val=(like_user_wwid,)
							catlead_sec_mail_rs = execute_query(sql,val)

							if catlead_sec_mail_rs != ():
								email_list.append(catlead_sec_mail_rs[0][0])

	if(state == 2):
		sql = "SELECT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1,ComponentType C1,ScheduleTableComponent S1, ScheduleStatusType S2 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND S1.ScheduleStatusID IN (2,3,6,NULL) ORDER BY FIELD(S1.ScheduleStatusID, 3, 6, 2, 1, 4, 5, 7),C1.ComponentName"
		val = (boardid,"yes")
		compids = execute_query(sql,val)
		
		status = ""
		status_interface = '<b><u>Interface list in Yet to Kickstart and Ongoing state: </u></b><br><br><table style="width: 100%;border: 1px solid #ddd;padding: 2px;">'
		status_interface += '<tr><td style="width: 35%;border-bottom: 1px solid #ddd;"><b>Interface</b></td><td style="width: 20%;border-bottom: 1px solid #ddd;"><b>Status</b></td><td style="width: 45%;border-bottom: 1px solid #ddd;"><b>Primary Electrical Owner</b></td></tr>'

		for q in compids:
			
			sql = "SELECT SecondaryWWID from ComponentReview C2 WHERE  C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s"
			val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],)
			sec_ele_wwid = execute_query(sql,val)

			sec_wwid = []
			if sec_ele_wwid != ():
				for i in sec_ele_wwid:
					if i[0] is not None:
						if (i[0] != []) and (i[0] != ['']):
							sec_wwid = i[0][1:-1].split(', ')

							for j in range(0,len(sec_wwid)):
								like_user_wwid = '%' + str(sec_wwid[j]) + '%'
								if sec_wwid[j] not in ['99999999',99999999]:
									sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
									val=(like_user_wwid,)
									email_id_rs = execute_query(sql,val)

									if email_id_rs != ():
										email_list.append(email_id_rs[0][0])

			sql = "select SecondaryWWID from ComponentDesign C2 WHERE  C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s"
			val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],)
			sec_des_wwid = execute_query(sql,val)

			des_wwid = []
			if sec_des_wwid != ():
				for i in sec_des_wwid:
					if i[0] is not None:
						if (i[0] != []) and (i[0] != ['']):
							des_wwid = i[0][1:-1].split(', ')

							for j in range(0,len(des_wwid)):
								like_user_wwid = '%' + str(des_wwid[j]) + '%'
								if des_wwid[j] not in ['99999999',99999999]:
									sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
									val=(like_user_wwid,)
									email_id_rs = execute_query(sql,val)

									if email_id_rs != ():
										email_list.append(email_id_rs[0][0])


			sql = "SELECT H1.EmailID FROM HomeTable H1,ComponentDesign C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
			val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],)
			primary_rev = execute_query(sql,val)

			sql = "SELECT H1.EmailID,IFNULL(H1.UserName,'') FROM HomeTable H1,ComponentReview C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID "
			val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],)
			primary_des = execute_query(sql,val)

			font_color = ''
			if q[3] == 1:	# signed-off
				font_color = 'green'
			elif q[3] == 2:	# ongoing
				font_color = 'orange'
			elif q[3] == 3:	# yet to kickstart
				font_color = 'red'

			if primary_des != ():
				status_interface += '<tr><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">'+q[1] + '</td><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">' + q[2] + '</td><td style="border-bottom: 1px solid #ddd;">' + primary_des[0][1] + '</td></tr>'

				if q[3] not in [1,'1',7,'7']:
					user_specific_comp_name.append([str(primary_des[0][0]),str(primary_des[0][1]),str(q[1])])

			else:
				status_interface += '<tr><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">'+q[1] + '</td><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">' + q[2] + '</td><td style="border-bottom: 1px solid #ddd;">-</td></tr>'

			sql = "SELECT H.EmailID FROM HomeTable H WHERE H.RoleID = 14"
			mgmt = execute_query_sql(sql)
				
			for j in primary_rev:
				email_list.append(j[0])

			for j in primary_des:
				email_list.append(j[0])

			for j in mgmt:
				for k in range(len(j)):
					email_list.append(j[k])
			status = status + q[1] + "&nbsp &nbsp &nbsp" + q[2] + "<br>"	

				
		d1 = str(cal[0][2])	
		email_list = sorted(set(email_list), reverse=True)

		status_interface += "</table>"

		for i in email_list:
			
			x=0
			temp_user_specific_msg = ''
			message_temp = ''

			for j in range(len(user_specific_comp_name)):
				if user_specific_comp_name[j][0] == i:
					if x==0:
						temp_user_specific_msg += '<font style="background-color:yellow;"><u>Interfaces to be reviewed/Signed-off by <b>'+user_specific_comp_name[j][1]+':</b></u></font><br>'
						temp_user_specific_msg += user_specific_comp_name[j][2]
					else:
						temp_user_specific_msg += '<br>'+user_specific_comp_name[j][2]
					x+=1

			message_temp += message + temp_user_specific_msg + '<br><br>'
			message_temp += status_interface
			message_temp += '<br><br>Thanks,<br>ERAM.'

			send_mail_html(i,subject,message_temp,email_list)	

	elif(state == 3):

		d1 = str(cal[0][1])	

		message += '<br><br>Thanks,<br>ERAM.'

		email_list = sorted(set(email_list), reverse=True)
		for i in email_list:
			send_mail(i,subject,message,email_list)

	#return redirect('/design_summary_data_page?boardid='+str(boardid))
	return redirect(url_for("design_summary_data_page",boardid=str(boardid),_external=True))

@app.route("/kickoff_2",methods = ['POST', 'GET'])
def kickoff_2():
	boardid = request.form.get("boardid")
	subject = request.form.get("subject")
	message = request.form.get("message")
	email_list = []

	wwid = session.get('wwid')

	# log table
	try:
		log_notes = 'User has triggerred Kickoff design for Design ID: '+str(boardid)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
		val = ('KickOff',boardid,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")

	sql = "SELECT * FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	res = execute_query(sql,val)

	if(len(res) == 0):
		sql = "INSERT INTO ScheduleTable VALUES (%s,2)"
		val = (boardid,)
		execute_query(sql,val)
	else:
		sql = "UPDATE ScheduleTable SET ScheduleStatusID = 2 WHERE BoardID = %s"
		val = (boardid,)
		execute_query(sql,val)


	date = datetime.datetime.now(tz).strftime('%Y-%m-%d')
	sql = "UPDATE DesignCalendar SET ActualStartDate = %s, ProposedStartDate = %s, BoardState = %s WHERE BoardID = %s"
	val = (date,date,'Design Review In-Progress',boardid)
	execute_query(sql,val)	

	# remove interfaces which are not valid
	remove_invalid_interfaces(boardid=boardid)

	sql = "SELECT BoardName, ClosureComment FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	result = execute_query(sql,val)
	boardname = result[0][0]
	closure_comment = result[0][1]

	sql = "SELECT ProposedEndDate FROM DesignCalendar WHERE BoardID =%s"
	val = (boardid,)
	enddate = execute_query(sql,val)
	EndDate=[]
	EndDate.append(enddate[0][0])
	if enddate[0][0]:
		ww_end_date = ' (WW' + str(get_isocalendar(enddate[0][0])[1]) + '.' + str(get_isocalendar(enddate[0][0])[2]) + ')'
	else:
		ww_end_date = ''

	EndDate.append(ww_end_date)

	sql = "SELECT a.BoardName,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,c.Username,DesignManagerWWID,d.Username,e.Username,core,IFNULL(f.RequestID,'') FROM BoardDetails a NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) LEFT JOIN HomeTable c ON a.DesignLeadWWID = c.WWID LEFT JOIN HomeTable d ON a.CADLeadWWID = d.WWID LEFT JOIN HomeTable e ON a.PIFLeadWWID = e.WWID LEFT JOIN RequestMap f ON a.BoardID=f.BoardID WHERE a.BoardID = %s"
	val = (boardid,)
	boarddeets = execute_query(sql,val)
	boarddeets_list = []
	for i in range(len(boarddeets)):
		role = boarddeets[len(boarddeets) - i - 1]
		boarddeets_list.append(role)

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
	val = (boardid,)
	designlist = execute_query(query,val)
	designlead_list = []
	for i in range(len(designlist)):
		role = designlist[len(designlist) - i - 1]
		designlead_list.append(role)
		eid = designlist[0][1]
		email_list.append(eid)


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.Username=b.DesignManagerWWID and b.BoardID AND b.BoardID = %s) LIMIT 1"
	val = (boardid,)
	designmanager = execute_query(query,val)
	if designmanager != ():
		email_list.append(designmanager[0][1])


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
	val = (boardid,)
	cadlist = execute_query(query,val)
	cadlead_list = []
	for i in range(len(cadlist)):
		role = cadlist[len(cadlist) - i - 1]
		cadlead_list.append(role)
		eid = cadlist[0][1]
		email_list.append(eid)


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.PIFLeadWWID and b.BoardID AND b.BoardID = %s)"
	piflist = execute_query(query,val)
	piflead_list = []
	for i in range(len(piflist)):
		role = piflist[len(piflist) - i - 1]
		piflead_list.append(role)
		eid = piflist[0][1]
		email_list.append(eid)

	# all pif leads
	email_list += get_pif_leads_email_id_by_board_id(boardid=boardid)

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)

	ww_date = []
	if cal[0][3]:
		cal_ww_start_date = ' (WW' + str(get_isocalendar(cal[0][3])[1]) + '.' + str(get_isocalendar(cal[0][3])[2]) + ')'
	else:
		cal_ww_start_date = ''
	ww_date.append(cal_ww_start_date)

	if cal[0][4]:
		cal_ww_end_date = ' (WW' + str(get_isocalendar(cal[0][4])[1]) + '.' + str(get_isocalendar(cal[0][4])[2]) + ')'
	else:
		cal_ww_end_date = ''
	ww_date.append(cal_ww_end_date)

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	sql = "SELECT DISTINCT B1.BoardID,B1.BoardID,B1.ComponentID,B1.PDG,B1.CommentDesigner,B1.PDG_Electrical,C1.ComponentName,S2.ScheduleTypeName,H1.UserName,H3.UserName,B1.CommentElectrical,H4.UserName,B1.CommentSignOffInterface,H5.UserName,B1.IsPdgElectricalSubmitted FROM BoardReviewDesigner B1, ComponentType C1, ScheduleTableComponent S1, ScheduleStatusType S2, ComponentReview C2, HomeTable H1, HomeTable H2, HomeTable H3, HomeTable H4, HomeTable H5 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID =%s AND B1.ComponentID = C2.ComponentID AND C2.PrimaryWWID = H1.WWID AND B1.CommentDesignUpdatedBy = H3.WWID AND B1.CommntElectricalUpdatedBy = H4.WWID AND B1.CommentSignOffInterfaceUpdateBy = H5.WWID ORDER BY FIELD(S1.ScheduleStatusID, 3, 6, 2, 1, 4, 5, 7), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
	val = (boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	comp = execute_query(sql,val)

	status = ""

	sql = "SELECT * FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	r = execute_query(sql,val)

	if(len(r)==0):
		status = "Yet to kickoff"
	else:
		sql = "SELECT S1.ScheduleTypeName FROM ScheduleStatusType S1, DesignCalendar D1, ScheduleTable S2 WHERE D1.BoardID = S2.BoardID AND S2.ScheduleStatusID = S1.ScheduleID AND D1.BoardID = %s "
		val = (boardid,)
		status = execute_query(sql,val)[0][0]	

	access_admin = 'no'	
	
	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	if(has_admin_access == 'yes'):
		access_admin = 'yes'

	sql = "SELECT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1,ComponentType C1,ScheduleTableComponent S1, ScheduleStatusType S2 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID ORDER BY FIELD(S1.ScheduleStatusID, 3, 6, 2, 1, 4, 5, 7),C1.ComponentName"
	val = (boardid,"yes")
	compids = execute_query(sql,val)	

	status_interface = '''<u><b>AR to Electrical Owner:</b></u><br>
	Please proceed to https://eram.apps1-fm-int.icloud.intel.com/ to submit feedback/sign-off. <br>
	To provide Feedback/Sign-Off : My Dashboard >> Feedback Submission Module >>  '''+boardname+'''<br><br>'''

	status_interface += '<b><u>Interface - Requested for Review: </u></b><br><br><table style="width: 100%;border: 1px solid #ddd;padding: 2px;">'
	status_interface += '<tr><td style="width: 35%;border-bottom: 1px solid #ddd;"><b>Interface</b></td><td style="width: 20%;border-bottom: 1px solid #ddd;"><b>Status</b></td><td style="width: 45%;border-bottom: 1px solid #ddd;"><b>Primary Electrical Owner</b></td></tr>'

	sql = "SELECT DISTINCT SecondaryWWID from ComponentReview C2, BoardReviewDesigner B1, CategoryLeadTable C3,ComponentType C4 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C2.ComponentID AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s  AND C2.CategoryID = C3.CategoryID AND C3.SKUID = %s AND C3.PlatformID = %s AND  C3.MemTypeID = %s AND C3.DesignTypeID = %s "
	val = (boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	sec_ele_wwid = execute_query(sql,val)

	sec_wwid = []

	if sec_ele_wwid != ():
		for i in sec_ele_wwid:
			if i[0] is not None:
				if (i[0] != []) and (i[0] != ['']):
					sec_wwid = i[0][1:-1].split(', ')

					for j in range(0,len(sec_wwid)):
						like_user_wwid = '%' + str(sec_wwid[j]) + '%'
						if sec_wwid[j] not in ['99999999',99999999]:
							sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
							val=(like_user_wwid,)
							email_id_rs = execute_query(sql,val)

							if email_id_rs != ():
								email_list.append(email_id_rs[0][0])

	sql = "SELECT DISTINCT SecondaryWWID FROM ComponentDesign C2,BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C2.ComponentID AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s"
	val = (boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	sec_des_wwid = execute_query(sql,val)

	des_wwid = []
	if sec_des_wwid != ():
		for i in sec_des_wwid:
			if i[0] is not None:
				if (i[0] != []) and (i[0] != ['']):
					des_wwid = i[0][1:-1].split(', ')

					for j in range(0,len(des_wwid)):
						like_user_wwid = '%' + str(des_wwid[j]) + '%'
						if des_wwid[j] not in ['99999999',99999999]:
							sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
							val=(like_user_wwid,)
							email_id_rs = execute_query(sql,val)

							if email_id_rs != ():
								email_list.append(email_id_rs[0][0])

	sql = "SELECT H.EmailID FROM HomeTable H WHERE H.RoleID = 14"
	mgmt = execute_query_sql(sql)

	for j in mgmt:
		for k in range(len(j)):
			email_list.append(j[k])

	user_specific_comp_name = []

	for q in compids:
		sql = "SELECT DISTINCT H1.EmailID FROM HomeTable H1, ComponentDesign C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
		val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		primary_rev = execute_query(sql,val)

		sql = "SELECT DISTINCT H1.EmailID,IFNULL(H1.UserName,'') FROM HomeTable H1, ComponentReview C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
		val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		primary_des = execute_query(sql,val)

		if primary_des != ():
			status_interface += '<tr><td style="border-bottom: 1px solid #ddd;">'+q[1] + '</td><td style="border-bottom: 1px solid #ddd;">' + q[2] + '</td><td style="border-bottom: 1px solid #ddd;">' + primary_des[0][1] + '</td></tr>'

			if q[3] not in [1,'1',7,'7']:
				user_specific_comp_name.append([str(primary_des[0][0]),str(primary_des[0][1]),str(q[1])])

		else:
			status_interface += '<tr><td style="border-bottom: 1px solid #ddd;">'+q[1] + '</td><td style="border-bottom: 1px solid #ddd;">' + q[2] + '</td><td style="border-bottom: 1px solid #ddd;">-</td></tr>'

		for j in primary_rev:
			email_list.append(j[0])
			
		for j in primary_des:
			email_list.append(j[0])
			
		
	catlead_sec_mail = []
	sql="SELECT DISTINCT a.CategoryName,b.EmailID,C3.CategoryLeadWWID1 from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2,BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PLatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID AND C2.ComponentID = B1.ComponentID ORDER BY cr.ComponentID"
	val=(boardid,"yes",sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	catlead=execute_query(sql,val)

	if(catlead != ()):
		for i in catlead:
			email_list.append(i[1])

			if i[2] is not None:
				if i[2] != []:
					cat_sec_wwid_list = i[2][1:-1].split(', ')

					for j in range(0,len(cat_sec_wwid_list)):
						like_user_wwid = '%' + str(cat_sec_wwid_list[j]) + '%'
						if cat_sec_wwid_list[j] not in ['99999999',99999999]:
							sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
							val=(like_user_wwid,)
							catlead_sec_mail_rs = execute_query(sql,val)

							if catlead_sec_mail_rs != ():
								email_list.append(catlead_sec_mail_rs[0][0])
					

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val=('yes',)
	admin_list = execute_query(sql,val)				

	for j in admin_list:
		email_list.append(j[0])

	sql = "SELECT EmailID FROM HomeTable WHERE RoleID = %s"
	val=(14,)
	admin_list1 = execute_query(sql,val)				

	for j in admin_list1:
		email_list.append(j[0])

	usernames = request.form.getlist("usernames")
	eids = []
	if(usernames):
		sql = "select EmailID from HomeTable where WWID in %s"
		val = (usernames,)
		eids = execute_query(sql, val)

	for i in range(len(eids)):
		if(eids[i][0]):
			email_list.append(eids[i][0])	



	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	rid = execute_query(sql,val)[0][0]

	timeline = ""
	if(rid == 1):
		timeline = "Core/Placement layout review Rev0p6"
	else:
		timeline = "Full layout review Rev1p0"


	status_interface += "</table>"

	d1 = str(cal[0][2])	
	email_list = sorted(set(email_list), reverse=True)
	
	for i in email_list:
		
		x=0
		temp_user_specific_msg = ''
		message_temp = ''
		for j in range(len(user_specific_comp_name)):
			if user_specific_comp_name[j][0] == i:
				if x==0:
					temp_user_specific_msg += '<font style="background-color:yellow;"><u>Interfaces to be reviewed/Signed-off by <b>'+user_specific_comp_name[j][1]+':</b></u></font><br>'
					temp_user_specific_msg += user_specific_comp_name[j][2]
				else:
					temp_user_specific_msg += '<br>'+user_specific_comp_name[j][2]
				x+=1

		message_temp = message + temp_user_specific_msg + '<br><br>'
		message_temp += status_interface
		message_temp += '<br><br>Thanks,<br>ERAM.'

		#send_mail(i,subject,message_temp,email_list)
		send_mail_html(i,subject,message_temp,email_list)


	# to freeze electrical and design owners for the Interface during Signoff Designs, incase in future ti avoid impact if we change owners for the interface
	sql="SELECT BoardID,PlatformID,SKUID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	bdetails = execute_query(sql,val)

	for row in bdetails:

		sql="SELECT ComponentID,CategoryID,PlatformID,SKUID,MemTypeID,DesignTypeID,PrimaryWWID,SecondaryWWID FROM ComponentReview WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s ORDER BY ComponentID ASC"
		val = (row[1],row[2],row[3],row[4])
		compdetails = execute_query(sql,val)

		if compdetails != ():

			for comp_row in compdetails:

				sql="SELECT CategoryLeadWWID,CategoryLeadWWID1 FROM CategoryLeadTable WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s AND CategoryID = %s"
				val = (row[1],row[2],row[3],row[4],comp_row[1])
				categorydetails = execute_query(sql,val)

				prim_cad_lead = 99999999
				sec_cad_lead = '[]'

				if categorydetails != ():
					prim_cad_lead = categorydetails[0][0]
					sec_cad_lead = categorydetails[0][1]

				sql = "INSERT INTO DesignElectricalOwners VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s) ON DUPLICATE KEY UPDATE PrimaryCategoryLead = %s, SecondaryCategoryLead = %s, PrimaryElectricalOwner = %s, SecondaryElectricalOwner = %s"
				val = (row[0],comp_row[0],comp_row[1],comp_row[2],comp_row[3],comp_row[4],comp_row[5],prim_cad_lead,sec_cad_lead,comp_row[6],comp_row[7],prim_cad_lead,sec_cad_lead,comp_row[6],comp_row[7])
				execute_query(sql, val)


		sql="SELECT ComponentID,CategoryID,PlatformID,SKUID,MemTypeID,DesignTypeID,PrimaryWWID,SecondaryWWID FROM ComponentDesign WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s ORDER BY ComponentID ASC"
		val = (row[1],row[2],row[3],row[4])
		compdetails2 = execute_query(sql,val)

		if compdetails2 != ():

			for comp_row in compdetails2:

				sql = "INSERT INTO DesignDesignOwners VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s) ON DUPLICATE KEY UPDATE PrimaryOwner = %s, SecondaryOwner = %s"
				val = (row[0],comp_row[0],comp_row[1],comp_row[2],comp_row[3],comp_row[4],comp_row[5],comp_row[6],comp_row[7],comp_row[6],comp_row[7])
				execute_query(sql, val)

	#return redirect('/design_summary_data_page?boardid='+str(boardid))
	return redirect(url_for("design_summary_data_page",boardid=str(boardid),_external=True))


@app.route("/kickoff",methods = ['POST', 'GET'])
def kickoff():
	boardid = request.form.get("boardid")
	email_list = []
	date = datetime.datetime.now(tz).strftime('%Y-%m-%d')
	

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	boardname = execute_query(sql,val)[0][0]

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)

	status = ""

	sql = "SELECT * FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	r = execute_query(sql,val)

	wwid = session.get('wwid')
	
	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	rid = execute_query(sql,val)[0][0]

	timeline = ""
	if(rid == 1):
		timeline = "Core/Placement layout review Rev0p6"
	else:
		timeline = "Full layout review Rev1p0"

			
	d1 = get_work_week_date_fmt(cal[0][2])	
	email_list = sorted(set(email_list), reverse=True)

	#subject= "[ID:"+boardid+"] "+boardname +" review request. Please Provide Feedback By "+get_work_week_date_fmt(cal[0][2])
	subject= "[ID:"+boardid+"] Review request. Please Provide Feedback By "+get_work_week_date_fmt(cal[0][2])
	message=''' Hello All, <br><br>

	This is a '''+timeline+''' request for ''' + boardname + ''' <br>

	Please provide the feedback / sign-off by <font style="color:red">''' + d1 +'''</font><br><br>'''

	sql = "SELECT UserName,WWID FROM HomeTable WHERE IsActive = 1 ORDER BY UserName"
	usernames = execute_query_sql(sql)
	user_list = []
	for i in usernames:
		user_list.append(i)

	clist = {}
	clist["message"]=message
	clist["subject"]=subject
	clist["boardname"]=boardname
	clist["usernames"]=user_list	
	return jsonify(clist)

@app.route("/no_signoff_popup",methods = ['POST', 'GET'])
def no_signoff_popup():
	boardid = request.form.get("boardid")
	email_list = []
	date = datetime.datetime.now(tz).strftime('%Y-%m-%d')

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	try:
		boardname = execute_query(sql,val)[0][0]
	except:
		boardname = ''

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)
	
	status = ""

	sql = "SELECT * FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	r = execute_query(sql,val)
	
	wwid = session.get('wwid')

	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	try:
		rid = execute_query(sql,val)[0][0]
	except:
		rid = 1

	timeline = ""
	if(rid == 1):
		timeline = "Core/Placement layout review Rev0p6"
	else:
		timeline = "Full layout review Rev1p0"

	subject= "[ID:"+boardid+"] No ERAM Sign-off"
	message=''' Hello All, <br><br>

	'''+timeline+''' for ''' +  boardname + ''' is not Signed Off by ERAM.<br><br>'''
	
	sql = "SELECT UserName,WWID FROM HomeTable WHERE IsActive = 1 ORDER BY UserName"
	usernames = execute_query_sql(sql)
	user_list = []
	for i in usernames:
		user_list.append(i)

	clist = {}
	clist["message"]=message
	clist["subject"]=subject
	clist["boardname"]=boardname
	clist["usernames"]=user_list	
	return jsonify(clist)

@app.route("/reject_popup",methods = ['POST', 'GET'])
def reject_popup():
	boardid = request.form.get("boardid")
	email_list = []
	date = datetime.datetime.now(tz).strftime('%Y-%m-%d')

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	boardname = execute_query(sql,val)[0][0]

	wwid = session.get('wwid')

	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	rid = execute_query(sql,val)[0][0]

	timeline = ""
	if(rid == 1):
		timeline = "Core/Placement layout review Rev0p6"
	else:
		timeline = "Full layout review Rev1p0"

	subject= "[ID:"+boardid+"] is Dropped / Rejected"
	message=''' Hello All, <br><br>

	The  '''+timeline+''' request for ''' + boardname + ''' is dropped / rejected.<br> <br>	

	Please Ignore the review request. <br><br>'''

	sql = "SELECT UserName,WWID FROM HomeTable WHERE IsActive = 1 ORDER BY UserName"
	usernames = execute_query_sql(sql)
	user_list = []
	for i in usernames:
		user_list.append(i)

	clist = {}
	clist["message"]=message
	clist["subject"]=subject
	clist["boardname"]=boardname
	clist["usernames"]=user_list	
	return jsonify(clist)

@app.route("/reminder_popup",methods = ['POST', 'GET'])
def reminder_popup():
	boardid = request.form.get("boardid")
	email_list = []
	date = datetime.datetime.now(tz).strftime('%Y-%m-%d')

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	try:
		boardname = execute_query(sql,val)[0][0]
	except:
		boardname = ''

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)
	
	try:
		d1 = get_work_week_date_fmt(cal[0][1])
		d2 = get_work_week_date_fmt(cal[0][2])
	except:
		d1 = ''
		d2 = ''

	sql = "SELECT ScheduleStatusID FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	try:
		state = execute_query(sql,val)[0][0]
	except:
		state = 2

	wwid = session.get('wwid')

	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	try:
		rid = execute_query(sql,val)[0][0]
	except:
		rid = 1

	timeline = ""
	if(rid == 1):
		timeline = "Core/Placement layout review Rev0p6"
	else:
		timeline = "Full layout review Rev1p0"

	subject = ''
	message = ''

	if(state == 2):
		status_interface = ""	

		subject= "[ID:"+boardid+"] Reminder for review request. Please Provide Feedback By " + d2
		message=''' Gentle Reminder, <br><br>

		This is a '''+timeline+''' request for ''' + boardname + ''' <br>

		Please provide the feedback / sign-off by <font style="color:red">''' + d2 +'''</font><br>  
		

		Please proceed to visit https://eram.apps1-fm-int.icloud.intel.com/ to submit your feedback/signoff. <br><br>'''


	elif(state == 3):

		subject= "[ID: "+boardid+"] Reminder for review request. Please Update Details By " + d1
		message=''' Gentle Reminder, <br><br>

		Start Date of '''+timeline+''' request for <b>''' + boardname + '''</b> is <b>'''+ d1 +'''</b> <br><br>

		<u><b> AR To Design/Layout Lead </b></u> <br>
		Please proceed to https://eram.apps1-fm-int.icloud.intel.com/ to submit feedback/sign-off. <br>
		To provide Feedback/Sign-Off : My Dashboard >> Feedback Submission Module >> '''+boardname+'''<br><br>'''


	sql = "SELECT UserName,WWID FROM HomeTable WHERE IsActive = 1 ORDER BY UserName"
	usernames = execute_query_sql(sql)
	user_list = []
	for i in usernames:
		user_list.append(i)

	clist = {}
	clist["message"]=message
	clist["subject"]=subject
	clist["boardname"]=boardname
	clist["usernames"]=user_list	
	return jsonify(clist)

@app.route("/signoff_2",methods = ['POST', 'GET'])
def signoff_2(is_automation=False,boardid=0):

	if not is_automation:
		boardid = request.form.get("boardid")
		comment = request.form.get("closure_comment_signoff")
		subject = request.form.get("subject")
		message = request.form.get("message")

	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	rid = execute_query(sql,val)[0][0]

	# mark all ongoing interface status to closed
	if(rid == 1):
		sql = "UPDATE ScheduleTableComponent SET ScheduleStatusID = %s WHERE BoardID = %s AND ScheduleStatusID = %s"
		val = (1,boardid,2)
		execute_query(sql,val)


	if is_automation:
		subject = ""
		message = ""
		comment = ""

		sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
		val = (boardid,)
		boardname = execute_query(sql,val)[0][0]

		if(rid == 1):
			comment = "Design closed for core/placement reviews."
			timeline = "Core/Placement layout review Rev0p6"

			subject = "[ID: "+boardid+"] is Closed"
			message =''' Hello All, <br><br>

			 '''+timeline+''' for ''' + boardname+ ''' is closed by ERAM. <br><br>'''

		else:
			comment = "Signed-Off"
			timeline = "Full layout review Rev1p0"

			subject = "[ID: "+boardid+"] is Signed-Off"
			message =''' Hello All, <br><br>

			 '''+timeline+''' for ''' + boardname+ ''' is signed-off by ERAM. <br><br>'''
		
	
	print("boardid final: ",boardid)
	email_list = []

	message += '<br><b>Comments: </b>' + comment + '<br><br>'

	wwid = session.get('wwid')
	
	# log table
	try:
		if is_automation:
			log_notes = 'Design Signed-Off by automation for Design ID: '+str(boardid)+'<br>Comments: '+str(comment)
		else:
			log_notes = 'User has Signed-Off Design for Design ID: '+str(boardid)+'<br>Comments: '+str(comment)

		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
		val = ('Signed-Off',boardid,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")

	sql = "SELECT * FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	res = execute_query(sql,val)
	email_list = []

	if(res == ()):
		sql = "INSERT INTO ScheduleTable VALUES (%s,1)"
		val = (boardid,)
		execute_query(sql,val)
	else:
		sql = "UPDATE ScheduleTable SET ScheduleStatusID = 1 WHERE BoardID = %s"
		val = (boardid,)
		execute_query(sql,val)


	date = datetime.datetime.now(tz).strftime('%Y-%m-%d')
	sql = "UPDATE DesignCalendar SET ActualEndDate = %s, ProposedEndDate = %s, BoardState = %s WHERE BoardID = %s"
	val = (date,date,'Design Signed-off',boardid)
	execute_query(sql,val)

	sql = "UPDATE BoardDetails SET ClosureComment = %s WHERE BoardID = %s"
	val = (comment,boardid)
	execute_query(sql,val)

	# delete all saved feedbacks during design closure
	#sql = "DELETE FROM BoardReview WHERE BoardID = %s AND (Submitted IS NULL OR Submitted <> %s)"
	#val = (boardid,'yes')
	#execute_query(sql,val)

	sql = "DELETE FROM BoardReview WHERE HasChild IS NULL AND BoardID = %s AND (Submitted IS NULL OR Submitted <> %s)"
	val = (boardid,"yes")
	execute_query(sql,val)

	# delete all uploaded files but not submitted feedbacks
	sql = "DELETE FROM UploadSignOffFilesTemp WHERE BoardID = %s"
	val = (boardid,)
	execute_query(sql,val)

	sql = "UPDATE BoardReview SET SignedOff_Reviewer2 = %s, Submitted_Reviewer2 = %s, is_edit_save_flag_design = %s WHERE BoardID = %s"
	val = ('yes','yes',0,boardid)
	execute_query(sql,val)

	# for Rev0p6 design, all saved feedbacks of "to be filled by design team" section to changed as submitted mode.
	if(rid == 1):
		sql = "UPDATE BoardReview SET DesignerFeedbackGiven = %s, Submitted_Designer = %s WHERE BoardID = %s AND Saved_Designer = %s"
		val = ('yes','yes',boardid,"yes")
		execute_query(sql,val)

	# remove interfaces which are not valid
	remove_invalid_interfaces(boardid=boardid)

	sql = "SELECT BoardName, ClosureComment FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	result = execute_query(sql,val)
	boardname = result[0][0]
	closure_comment = result[0][1]

	sql = "SELECT ProposedEndDate FROM DesignCalendar WHERE BoardID =%s"
	val = (boardid,)
	enddate = execute_query(sql,val)
	EndDate=[]
	EndDate.append(enddate[0][0])

	if enddate[0][0]:
		ww_end_date = ' (WW' + str(get_isocalendar(enddate[0][0])[1]) + '.' + str(get_isocalendar(enddate[0][0])[2]) + ')'
	else:
		ww_end_date = ''

	EndDate.append(ww_end_date)

	sql = "SELECT a.BoardName,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,c.Username,DesignManagerWWID,d.Username,e.Username,core,IFNULL(f.RequestID,'') FROM BoardDetails a NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) LEFT JOIN HomeTable c ON a.DesignLeadWWID = c.WWID LEFT JOIN HomeTable d ON a.CADLeadWWID = d.WWID LEFT JOIN HomeTable e ON a.PIFLeadWWID = e.WWID LEFT JOIN RequestMap f ON a.BoardID=f.BoardID WHERE a.BoardID = %s"
	val = (boardid,)
	boarddeets = execute_query(sql,val)
	boarddeets_list = []
	for i in range(len(boarddeets)):
		role = boarddeets[len(boarddeets) - i - 1]
		boarddeets_list.append(role)

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
	val = (boardid,)
	designlist = execute_query(query,val)
	designlead_list = []
	for i in range(len(designlist)):
		role = designlist[len(designlist) - i - 1]
		designlead_list.append(role)
		eid = designlist[0][1]
		email_list.append(eid)


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
	val = (boardid,)
	cadlist = execute_query(query,val)
	cadlead_list = []
	for i in range(len(cadlist)):
		role = cadlist[len(cadlist) - i - 1]
		cadlead_list.append(role)
		eid = cadlist[0][1]
		email_list.append(eid)

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.Username=b.DesignManagerWWID and b.BoardID AND b.BoardID = %s) LIMIT 1"
	val = (boardid,)
	designmanager = execute_query(query,val)
	if designmanager != ():
		email_list.append(designmanager[0][1])

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
	val = (boardid,)
	cadlist = execute_query(query,val)
	cadlead_list = []
	for i in range(len(cadlist)):
		role = cadlist[len(cadlist) - i - 1]
		cadlead_list.append(role)
		eid = cadlist[0][1]
		email_list.append(eid)

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.PIFLeadWWID and b.BoardID AND b.BoardID = %s)"
	piflist = execute_query(query,val)
	piflead_list = []
	for i in range(len(piflist)):
		role = piflist[len(piflist) - i - 1]
		piflead_list.append(role)
		eid = piflist[0][1]
		email_list.append(eid)

	# all pif leads
	email_list += get_pif_leads_email_id_by_board_id(boardid=boardid)

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)

	ww_date = []
	if cal[0][3]:
		cal_ww_start_date = ' (WW' + str(get_isocalendar(cal[0][3])[1]) + '.' + str(get_isocalendar(cal[0][3])[2]) + ')'
	else:
		cal_ww_start_date = ''
	ww_date.append(cal_ww_start_date)

	if cal[0][4]:
		cal_ww_end_date = ' (WW' + str(get_isocalendar(cal[0][4])[1]) + '.' + str(get_isocalendar(cal[0][4])[2]) + ')'
	else:
		cal_ww_end_date = ''
	ww_date.append(cal_ww_end_date)

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	sql = "SELECT DISTINCT B1.BoardID,B1.BoardID,B1.ComponentID,B1.PDG,B1.CommentDesigner,B1.PDG_Electrical,C1.ComponentName,S2.ScheduleTypeName,H1.UserName,H3.UserName,B1.CommentElectrical,H4.UserName,B1.CommentSignOffInterface,H5.UserName,B1.IsPdgElectricalSubmitted FROM BoardReviewDesigner B1, ComponentType C1, ScheduleTableComponent S1, ScheduleStatusType S2, ComponentReview C2, HomeTable H1, HomeTable H2, HomeTable H3, HomeTable H4, HomeTable H5 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID =%s AND B1.ComponentID = C2.ComponentID AND C2.PrimaryWWID = H1.WWID AND B1.CommentDesignUpdatedBy = H3.WWID AND B1.CommntElectricalUpdatedBy = H4.WWID AND B1.CommentSignOffInterfaceUpdateBy = H5.WWID ORDER BY FIELD(S1.ScheduleStatusID, 3, 6, 2, 1, 4, 5, 7), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
	val = (boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	comp = execute_query(sql,val)

	access_admin = 'no'	
	
	#wwid = session.get('wwid')
	#wwid = 11806709
	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	if(has_admin_access == 'yes'):
		access_admin = 'yes'

	sql = "SELECT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1,ComponentType C1,ScheduleTableComponent S1, ScheduleStatusType S2 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID ORDER BY FIELD(S1.ScheduleStatusID, 3, 6, 2, 1, 4, 5, 7),C1.ComponentName"
	val = (boardid,"yes")
	compids = execute_query(sql,val)

	status = ""
	status_interface = '<br><table style="width: 100%;border: 1px solid #ddd;padding: 2px;">'
	status_interface += '<tr><td style="width: 35%;border-bottom: 1px solid #ddd;"><b>Interface</b></td><td style="width: 20%;border-bottom: 1px solid #ddd;"><b>Status</b></td><td style="width: 45%;border-bottom: 1px solid #ddd;"><b>Primary Electrical Owner</b></td></tr>'
		
	sql = "SELECT DISTINCT SecondaryWWID from ComponentReview C2, BoardReviewDesigner B1, CategoryLeadTable C3,ComponentType C4 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C2.ComponentID AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s  AND C2.CategoryID = C3.CategoryID AND C3.SKUID = %s AND C3.PlatformID = %s AND  C3.MemTypeID = %s AND C3.DesignTypeID = %s "
	val = (boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	sec_ele_wwid = execute_query(sql,val)

	sec_wwid = []
	if sec_ele_wwid != ():
		for i in sec_ele_wwid:
			if i[0] is not None:
				if (i[0] != []) and (i[0] != ['']):
					sec_wwid = i[0][1:-1].split(', ')

					for j in range(0,len(sec_wwid)):
						like_user_wwid = '%' + str(sec_wwid[j]) + '%'
						if sec_wwid[j] not in ['99999999',99999999]:
							sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
							val=(like_user_wwid,)
							email_id_rs = execute_query(sql,val)

							if email_id_rs != ():
								email_list.append(email_id_rs[0][0])

	sql = "SELECT DISTINCT SecondaryWWID FROM ComponentDesign C2,BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C2.ComponentID AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s"
	val = (boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	sec_des_wwid = execute_query(sql,val)

	des_wwid = []
	if sec_des_wwid != ():
		for i in sec_des_wwid:
			if i[0] is not None:
				if (i[0] != []) and (i[0] != ['']):
					des_wwid = i[0][1:-1].split(', ')

					for j in range(0,len(des_wwid)):
						like_user_wwid = '%' + str(des_wwid[j]) + '%'
						if des_wwid[j] not in ['99999999',99999999]:
							sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
							val=(like_user_wwid,)
							email_id_rs = execute_query(sql,val)

							if email_id_rs != ():
								email_list.append(email_id_rs[0][0])

	sql = "SELECT H.EmailID FROM HomeTable H WHERE H.RoleID = 14"
	mgmt = execute_query_sql(sql)

	for j in mgmt:
		for k in range(len(j)):
			email_list.append(j[k])


	for q in compids:
		sql = "SELECT H1.EmailID FROM HomeTable H1, ComponentDesign C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
		val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		primary_rev = execute_query(sql,val)

		sql = "SELECT H1.EmailID,IFNULL(H1.UserName,'') FROM HomeTable H1, ComponentReview C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
		val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		primary_des = execute_query(sql,val)

		for j in primary_rev:
			email_list.append(j[0])
			
		for j in primary_des:
			email_list.append(j[0])


		font_color = ''
		comp_status_name = copy.deepcopy(q[2])
		if q[3] == 1:	# signed-off
			font_color = 'green'

			if rid == 1:
				comp_status_name = "Closed"

		elif q[3] == 2:	# ongoing
			font_color = 'orange'
		elif q[3] == 3:	# yet to kickstart
			font_color = 'red'

		if primary_des != ():
			status_interface += '<tr><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">'+q[1] + '</td><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">' + comp_status_name + '</td><td style="border-bottom: 1px solid #ddd;">' + primary_des[0][1] + '</td></tr>'
		else:
			status_interface += '<tr><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">'+q[1] + '</td><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">' + comp_status_name + '</td><td style="border-bottom: 1px solid #ddd;">-</td></tr>'
			
		status = status + q[1] + "&nbsp &nbsp &nbsp" + q[2] + "<br>"		

	status_interface += "</table>"

	message += status_interface

	message += '<br><br>Thanks,<br>ERAM.'

	catlead_sec_mail = []

	sql="SELECT DISTINCT a.CategoryName,b.EmailID,C3.CategoryLeadWWID1 from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2,BoardReviewDesigner B1 WHERE cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PLatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID AND C2.ComponentID = B1.ComponentID AND B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s ORDER BY cr.ComponentID"
	val=(sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],boardid,"yes")
	catlead=execute_query(sql,val)
	if(catlead != ()):
		for i in catlead:
			email_list.append(i[1])

			if i[2] is not None:
				if i[2] != []:
					cat_sec_wwid_list = i[2][1:-1].split(', ')

					for j in range(0,len(cat_sec_wwid_list)):
						like_user_wwid = '%' + str(cat_sec_wwid_list[j]) + '%'
						if cat_sec_wwid_list[j] not in ['99999999',99999999]:
							sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
							val=(like_user_wwid,)
							catlead_sec_mail_rs = execute_query(sql,val)

							if catlead_sec_mail_rs != ():
								email_list.append(catlead_sec_mail_rs[0][0])

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val=('yes',)
	admin_list = execute_query(sql,val)				

	for j in admin_list:
		email_list.append(j[0])

	usernames = request.form.getlist("usernames")
	eids = []
	if(usernames):
		sql = "select EmailID from HomeTable where WWID in %s"
		val = (usernames,)
		eids = execute_query(sql, val)

	for i in range(len(eids)):
		if(eids[i][0]):
			email_list.append(eids[i][0])	

			
	d1 = str(cal[0][2])	
	email_list = sorted(set(email_list), reverse=True)
		
	for i in email_list:
		send_mail(i,subject,message,email_list)

	# to freeze electrical and design owners for the Interface during Signoff Designs, incase in future t0 avoid impact if we change owners for the interface
	sql="SELECT BoardID,PlatformID,SKUID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	bdetails = execute_query(sql,val)

	for row in bdetails:

		sql="SELECT ComponentID,CategoryID,PlatformID,SKUID,MemTypeID,DesignTypeID,PrimaryWWID,SecondaryWWID FROM ComponentReview WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s ORDER BY ComponentID ASC"
		val = (row[1],row[2],row[3],row[4])
		compdetails = execute_query(sql,val)

		if compdetails != ():

			for comp_row in compdetails:

				sql="SELECT CategoryLeadWWID,CategoryLeadWWID1 FROM CategoryLeadTable WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s AND CategoryID = %s"
				val = (row[1],row[2],row[3],row[4],comp_row[1])
				categorydetails = execute_query(sql,val)

				prim_cad_lead = 99999999
				sec_cad_lead = '[]'

				if categorydetails != ():
					prim_cad_lead = categorydetails[0][0]
					sec_cad_lead = categorydetails[0][1]

				sql = "INSERT INTO DesignElectricalOwners VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s) ON DUPLICATE KEY UPDATE PrimaryCategoryLead = %s, SecondaryCategoryLead = %s, PrimaryElectricalOwner = %s, SecondaryElectricalOwner = %s"
				val = (row[0],comp_row[0],comp_row[1],comp_row[2],comp_row[3],comp_row[4],comp_row[5],prim_cad_lead,sec_cad_lead,comp_row[6],comp_row[7],prim_cad_lead,sec_cad_lead,comp_row[6],comp_row[7])
				execute_query(sql, val)


		sql="SELECT ComponentID,CategoryID,PlatformID,SKUID,MemTypeID,DesignTypeID,PrimaryWWID,SecondaryWWID FROM ComponentDesign WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s ORDER BY ComponentID ASC"
		val = (row[1],row[2],row[3],row[4])
		compdetails2 = execute_query(sql,val)

		if compdetails2 != ():

			for comp_row in compdetails2:

				sql = "INSERT INTO DesignDesignOwners VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s) ON DUPLICATE KEY UPDATE PrimaryOwner = %s, SecondaryOwner = %s"
				val = (row[0],comp_row[0],comp_row[1],comp_row[2],comp_row[3],comp_row[4],comp_row[5],comp_row[6],comp_row[7],comp_row[6],comp_row[7])
				execute_query(sql, val)

	if is_automation:
		return True

	#return redirect('/design_summary_data_page?boardid='+str(boardid))
	return redirect(url_for("design_summary_data_page",boardid=str(boardid),_external=True))

@app.route("/signoff",methods = ['POST', 'GET'])
def signoff():
	boardid = request.form.get("boardid")
	email_list = []
	clist = {}
	date = datetime.datetime.now(tz).strftime('%Y-%m-%d')
	

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	boardname = execute_query(sql,val)[0][0]

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)
	
	status = ""

	sql = "SELECT * FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	r = execute_query(sql,val)
	
	wwid = session.get('wwid')

	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	rid = execute_query(sql,val)[0][0]

	timeline = ""
	if(rid == 1):
		timeline = "Core/Placement layout review Rev0p6"
		d1 = str(cal[0][1])	
		email_list = sorted(set(email_list), reverse=True)

		clist["comments"] = "Design closed for core/placement reviews."

		subject= "[ID: "+boardid+"] is Closed"
		message=''' Hello All, <br><br>

		 '''+timeline+''' for ''' + boardname+ ''' is closed by ERAM. <br><br>'''

	else:
		timeline = "Full layout review Rev1p0"
		d1 = str(cal[0][1])	
		email_list = sorted(set(email_list), reverse=True)

		clist["comments"] = "Signed-Off"

		subject= "[ID: "+boardid+"] is Signed-Off"
		message=''' Hello All, <br><br>

		 '''+timeline+''' for ''' + boardname+ ''' is signed-off by ERAM. <br><br>'''


	sql = "SELECT UserName,WWID FROM HomeTable WHERE IsActive = 1 ORDER BY UserName"
	usernames = execute_query_sql(sql)
	user_list = []
	for i in usernames:
		user_list.append(i)

	clist["message"]=message
	clist["subject"]=subject
	clist["boardname"]=boardname
	clist["usernames"]=user_list

	return jsonify(clist)

@app.route("/reject2",methods = ['POST', 'GET'])
def reject2():
	boardid = request.form.get("boardid")
	comment = request.form.get("closure_comment")
	subject = request.form.get("subject")
	message = request.form.get("message")
	usernames = request.form.getlist("usernames")

	message += '<br>Comments: ' + comment + '<br><br>Thanks,<br>ERAM.'

	wwid = session.get('wwid')

	# log table
	try:
		log_notes = 'User has Rejected Design ID: '+str(boardid)+'<br>Comments: '+str(comment)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
		val = ('Rejected',boardid,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")

	sql = "SELECT * FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	res = execute_query(sql,val)
	email_list = []
	if(res == ()):
		sql = "INSERT INTO ScheduleTable VALUES (%s,7)"
		val = (boardid,)
		execute_query(sql,val)
	else:
		sql = "UPDATE ScheduleTable SET ScheduleStatusID = 7 WHERE BoardID = %s"
		val = (boardid,)
		execute_query(sql,val)

	sql = "UPDATE BoardDetails SET ClosureComment = %s WHERE BoardID = %s"
	val = (comment,boardid)
	execute_query(sql,val)

	date = datetime.datetime.now(tz).strftime('%Y-%m-%d')
	sql = "UPDATE DesignCalendar SET ActualEndDate = %s, ProposedEndDate = %s, BoardState = %s WHERE BoardID = %s"
	val = (date,date,'Design Not signed-off',boardid)
	execute_query(sql,val)

	# delete all saved feedbacks during design closure
	sql = "DELETE FROM BoardReview WHERE BoardID = %s AND (Submitted IS NULL OR Submitted <> %s)"
	val = (boardid,'yes')
	execute_query(sql,val)

	# delete all uploaded files but not submitted feedbacks
	sql = "DELETE FROM UploadSignOffFilesTemp WHERE BoardID = %s"
	val = (boardid,)
	execute_query(sql,val)

	# remove interfaces which are not valid
	remove_invalid_interfaces(boardid=boardid)

	sql = "SELECT BoardName, ClosureComment FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	result = execute_query(sql,val)
	boardname = result[0][0]
	closure_comment = result[0][1]

	sql = "SELECT ProposedEndDate FROM DesignCalendar WHERE BoardID =%s"
	val = (boardid,)
	enddate = execute_query(sql,val)
	EndDate=[]
	EndDate.append(enddate[0][0])

	if enddate[0][0]:
		ww_end_date = ' (WW' + str(get_isocalendar(enddate[0][0])[1]) + '.' + str(get_isocalendar(enddate[0][0])[2]) + ')'
	else:
		ww_end_date = ''

	EndDate.append(ww_end_date)

	sql = "SELECT a.BoardName,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,c.Username,DesignManagerWWID,d.Username,e.Username,core,IFNULL(f.RequestID,'') FROM BoardDetails a NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) LEFT JOIN HomeTable c ON a.DesignLeadWWID = c.WWID LEFT JOIN HomeTable d ON a.CADLeadWWID = d.WWID LEFT JOIN HomeTable e ON a.PIFLeadWWID = e.WWID LEFT JOIN RequestMap f ON a.BoardID=f.BoardID WHERE a.BoardID = %s"
	val = (boardid,)
	boarddeets = execute_query(sql,val)
	boarddeets_list = []
	for i in range(len(boarddeets)):
		role = boarddeets[len(boarddeets) - i - 1]
		boarddeets_list.append(role)

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
	val = (boardid,)
	designlist = execute_query(query,val)
	designlead_list = []
	for i in range(len(designlist)):
		role = designlist[len(designlist) - i - 1]
		designlead_list.append(role)
		eid = designlist[0][1]
		email_list.append(eid)

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.Username=b.DesignManagerWWID and b.BoardID AND b.BoardID = %s) LIMIT 1"
	val = (boardid,)
	designmanager = execute_query(query,val)
	if designmanager != ():
		email_list.append(designmanager[0][1])


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
	val = (boardid,)
	cadlist = execute_query(query,val)
	cadlead_list = []
	for i in range(len(cadlist)):
		role = cadlist[len(cadlist) - i - 1]
		cadlead_list.append(role)
		eid = cadlist[0][1]
		email_list.append(eid)


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.PIFLeadWWID and b.BoardID AND b.BoardID = %s)"
	piflist = execute_query(query,val)
	piflead_list = []
	for i in range(len(piflist)):
		role = piflist[len(piflist) - i - 1]
		piflead_list.append(role)
		eid = piflist[0][1]
		email_list.append(eid)

	# all pif leads
	email_list += get_pif_leads_email_id_by_board_id(boardid=boardid)

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)

	ww_date = []
	if cal[0][3]:
		cal_ww_start_date = ' (WW' + str(get_isocalendar(cal[0][3])[1]) + '.' + str(get_isocalendar(cal[0][3])[2]) + ')'
	else:
		cal_ww_start_date = ''
	ww_date.append(cal_ww_start_date)

	if cal[0][4]:
		cal_ww_end_date = ' (WW' + str(get_isocalendar(cal[0][4])[1]) + '.' + str(get_isocalendar(cal[0][4])[2]) + ')'
	else:
		cal_ww_end_date = ''
	ww_date.append(cal_ww_end_date)

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	sql = "SELECT DISTINCT B1.BoardID,B1.BoardID,B1.ComponentID,B1.PDG,B1.CommentDesigner,B1.PDG_Electrical,C1.ComponentName,S2.ScheduleTypeName,H1.UserName,H3.UserName,B1.CommentElectrical,H4.UserName,B1.CommentSignOffInterface,H5.UserName,B1.IsPdgElectricalSubmitted FROM BoardReviewDesigner B1, ComponentType C1, ScheduleTableComponent S1, ScheduleStatusType S2, ComponentReview C2, HomeTable H1, HomeTable H2, HomeTable H3, HomeTable H4, HomeTable H5 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID =%s AND B1.ComponentID = C2.ComponentID AND C2.PrimaryWWID = H1.WWID AND B1.CommentDesignUpdatedBy = H3.WWID AND B1.CommntElectricalUpdatedBy = H4.WWID AND B1.CommentSignOffInterfaceUpdateBy = H5.WWID ORDER BY FIELD(S1.ScheduleStatusID, 3, 6, 2, 1, 4, 5, 7), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
	val = (boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	comp = execute_query(sql,val)
	
	status = ""

	sql = "SELECT * FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	r = execute_query(sql,val)

	if(len(r)==0):
		status = "Yet to kickoff"
	else:
		sql = "SELECT S1.ScheduleTypeName FROM ScheduleStatusType S1, DesignCalendar D1, ScheduleTable S2 WHERE D1.BoardID = S2.BoardID AND S2.ScheduleStatusID = S1.ScheduleID AND D1.BoardID = %s "
		val = (boardid,)
		status = execute_query(sql,val)[0][0]	

	access_admin = 'no'	
	

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	if(has_admin_access == 'yes'):
		access_admin = 'yes'
	
	sql = "SELECT H.EmailID FROM HomeTable H WHERE H.RoleID = 14"
	mgmt = execute_query_sql(sql)

	for j in mgmt:
		for k in range(len(j)):
			email_list.append(j[k])

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val=('yes',)
	admin_list = execute_query(sql,val)				

	for j in admin_list:
		email_list.append(j[0])



	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	rid = execute_query(sql,val)[0][0]

	timeline = ""
	if(rid == 1):
		timeline = "Core/Placement layout review Rev0p6"
	else:
		timeline = "Full layout review Rev1p0"

			
	d1 = str(cal[0][1])	

	eids = []
	if(usernames):
		sql = "select EmailID from HomeTable where WWID in %s"
		val = (usernames,)
		eids = execute_query(sql, val)

	for i in range(len(eids)):
		if(eids[i][0]):
			email_list.append(eids[i][0])	

	email_list = sorted(set(email_list), reverse=True)
	for i in email_list:
		send_mail(i,subject,message,email_list)
	#send_mail(reciever=', '.join(email_list),subject=subject,message=message)

	#return redirect('/design_summary_data_page?boardid='+str(boardid))
	return redirect(url_for("design_summary_data_page",boardid=str(boardid),_external=True))

@app.route("/nosignoff",methods = ['POST', 'GET'])
def nosignoff():
	boardid = request.form.get("boardid")
	comment = request.form.get("closure_comment")
	subject = request.form.get("subject")
	message = request.form.get("message")
	usernames = request.form.getlist("usernames")

	message += '<br>Comments: ' + comment + '<br><br>'

	wwid = session.get('wwid')

	# log table
	try:
		log_notes = 'User has triggerred No Signed-Off for Design ID: '+str(boardid)+'<br>Comments: '+str(comment)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
		val = ('No SignOff',boardid,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")

	sql = "SELECT * FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	res = execute_query(sql,val)
	email_list = []

	if(res == ()):
		sql = "INSERT INTO ScheduleTable VALUES (%s,5)"
		val = (boardid,)
		execute_query(sql,val)
	else:
		sql = "UPDATE ScheduleTable SET ScheduleStatusID = 5 WHERE BoardID = %s"
		val = (boardid,)
		execute_query(sql,val)

	sql = "UPDATE BoardDetails SET ClosureComment = %s WHERE BoardID = %s"
	val = (comment,boardid)
	execute_query(sql,val)

	date = datetime.datetime.now(tz).strftime('%Y-%m-%d')
	sql = "UPDATE DesignCalendar SET ActualEndDate = %s, ProposedEndDate = %s, BoardState = %s WHERE BoardID = %s"
	val = (date,date,'Design Not signed-off',boardid)
	execute_query(sql,val)

	# delete all saved feedbacks during design closure
	sql = "DELETE FROM BoardReview WHERE BoardID = %s AND (Submitted IS NULL OR Submitted <> %s)"
	val = (boardid,'yes')
	execute_query(sql,val)

	# delete all uploaded files but not submitted feedbacks
	sql = "DELETE FROM UploadSignOffFilesTemp WHERE BoardID = %s"
	val = (boardid,)
	execute_query(sql,val)

	# remove interfaces which are not valid
	remove_invalid_interfaces(boardid=boardid)

	sql = "SELECT BoardName, ClosureComment FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	result = execute_query(sql,val)
	boardname = result[0][0]
	closure_comment = result[0][1]

	sql = "SELECT ProposedEndDate FROM DesignCalendar WHERE BoardID =%s"
	val = (boardid,)
	enddate = execute_query(sql,val)
	EndDate=[]
	EndDate.append(enddate[0][0])

	if enddate[0][0]:
		ww_end_date = ' (WW' + str(get_isocalendar(enddate[0][0])[1]) + '.' + str(get_isocalendar(enddate[0][0])[2]) + ')'
	else:
		ww_end_date = ''

	EndDate.append(ww_end_date)

	sql = "SELECT a.BoardName,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,c.Username,DesignManagerWWID,d.Username,e.Username,core,IFNULL(f.RequestID,'') FROM BoardDetails a NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) LEFT JOIN HomeTable c ON a.DesignLeadWWID = c.WWID LEFT JOIN HomeTable d ON a.CADLeadWWID = d.WWID LEFT JOIN HomeTable e ON a.PIFLeadWWID = e.WWID LEFT JOIN RequestMap f ON a.BoardID=f.BoardID WHERE a.BoardID = %s"
	val = (boardid,)
	boarddeets = execute_query(sql,val)
	boarddeets_list = []
	for i in range(len(boarddeets)):
		role = boarddeets[len(boarddeets) - i - 1]
		boarddeets_list.append(role)

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
	val = (boardid,)
	designlist = execute_query(query,val)
	designlead_list = []
	for i in range(len(designlist)):
		role = designlist[len(designlist) - i - 1]
		designlead_list.append(role)
		eid = designlist[0][1]
		email_list.append(eid)

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.Username=b.DesignManagerWWID and b.BoardID AND b.BoardID = %s) LIMIT 1"
	val = (boardid,)
	designmanager = execute_query(query,val)
	if designmanager != ():
		email_list.append(designmanager[0][1])


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
	val = (boardid,)
	cadlist = execute_query(query,val)
	cadlead_list = []
	for i in range(len(cadlist)):
		role = cadlist[len(cadlist) - i - 1]
		cadlead_list.append(role)
		eid = cadlist[0][1]
		email_list.append(eid)


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.PIFLeadWWID and b.BoardID AND b.BoardID = %s)"
	piflist = execute_query(query,val)
	piflead_list = []
	for i in range(len(piflist)):
		role = piflist[len(piflist) - i - 1]
		piflead_list.append(role)
		eid = piflist[0][1]
		email_list.append(eid)

	# all pif leads
	email_list += get_pif_leads_email_id_by_board_id(boardid=boardid)

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)

	ww_date = []
	if cal[0][3]:
		cal_ww_start_date = ' (WW' + str(get_isocalendar(cal[0][3])[1]) + '.' + str(get_isocalendar(cal[0][3])[2]) + ')'
	else:
		cal_ww_start_date = ''
	ww_date.append(cal_ww_start_date)

	if cal[0][4]:
		cal_ww_end_date = ' (WW' + str(get_isocalendar(cal[0][4])[1]) + '.' + str(get_isocalendar(cal[0][4])[2]) + ')'
	else:
		cal_ww_end_date = ''
	ww_date.append(cal_ww_end_date)

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	sql = "SELECT DISTINCT B1.BoardID,B1.BoardID,B1.ComponentID,B1.PDG,B1.CommentDesigner,B1.PDG_Electrical,C1.ComponentName,S2.ScheduleTypeName,H1.UserName,H3.UserName,B1.CommentElectrical,H4.UserName,B1.CommentSignOffInterface,H5.UserName,B1.IsPdgElectricalSubmitted FROM BoardReviewDesigner B1, ComponentType C1, ScheduleTableComponent S1, ScheduleStatusType S2, ComponentReview C2, HomeTable H1, HomeTable H2, HomeTable H3, HomeTable H4, HomeTable H5 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID =%s AND B1.ComponentID = C2.ComponentID AND C2.PrimaryWWID = H1.WWID AND B1.CommentDesignUpdatedBy = H3.WWID AND B1.CommntElectricalUpdatedBy = H4.WWID AND B1.CommentSignOffInterfaceUpdateBy = H5.WWID ORDER BY FIELD(S1.ScheduleStatusID, 3, 6, 2, 1, 4, 5, 7), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
	val = (boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	comp = execute_query(sql,val)

	access_admin = 'no'	
	

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	if(has_admin_access == 'yes'):
		access_admin = 'yes'
	
	sql = "SELECT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1,ComponentType C1,ScheduleTableComponent S1, ScheduleStatusType S2 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID ORDER BY FIELD(S1.ScheduleStatusID, 3, 6, 2, 1, 4, 5, 7),C1.ComponentName"
	val = (boardid,"yes")
	compids = execute_query(sql,val)
	
	status = ""
	status_interface = '<br><table style="width: 100%;border: 1px solid #ddd;padding: 2px;">'
	status_interface += '<tr><td style="width: 35%;border-bottom: 1px solid #ddd;"><b>Interface</b></td><td style="width: 20%;border-bottom: 1px solid #ddd;"><b>Status</b></td><td style="width: 45%;border-bottom: 1px solid #ddd;"><b>Primary Electrical Owner</b></td></tr>'

	sql = "SELECT DISTINCT SecondaryWWID from ComponentReview C2, BoardReviewDesigner B1, CategoryLeadTable C3,ComponentType C4 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.IsPdgDesignSubmitted = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s  AND C2.CategoryID = C3.CategoryID AND C3.SKUID = %s AND C3.PlatformID = %s AND  C3.MemTypeID = %s AND C3.DesignTypeID = %s "
	val = (boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	sec_ele_wwid = execute_query(sql,val)

	sec_wwid = []
	if sec_ele_wwid != ():
		for i in sec_ele_wwid:
			if i[0] is not None:
				if (i[0] != []) and (i[0] != ['']):
					sec_wwid = i[0][1:-1].split(', ')

					for j in range(0,len(sec_wwid)):
						like_user_wwid = '%' + str(sec_wwid[j]) + '%'
						if sec_wwid[j] not in ['99999999',99999999]:
							sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
							val=(like_user_wwid,)
							email_id_rs = execute_query(sql,val)

							if email_id_rs != ():
								email_list.append(email_id_rs[0][0])

	sql = "SELECT DISTINCT SecondaryWWID FROM ComponentDesign C2,BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.IsPdgDesignSubmitted = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s"
	val = (boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	sec_des_wwid = execute_query(sql,val)

	des_wwid = []
	if sec_des_wwid != ():
		for i in sec_des_wwid:
			if i[0] is not None:
				if (i[0] != []) and (i[0] != ['']):
					des_wwid = i[0][1:-1].split(', ')

					for j in range(0,len(des_wwid)):
						like_user_wwid = '%' + str(des_wwid[j]) + '%'
						if des_wwid[j] not in ['99999999',99999999]:
							sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
							val=(like_user_wwid,)
							email_id_rs = execute_query(sql,val)

							if email_id_rs != ():
								email_list.append(email_id_rs[0][0])

	sql = "SELECT H.EmailID FROM HomeTable H WHERE H.RoleID = 14"
	mgmt = execute_query_sql(sql)

	for j in mgmt:
		for k in range(len(j)):
			email_list.append(j[k])

	for q in compids:
		sql = "SELECT H1.EmailID FROM HomeTable H1, ComponentDesign C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
		val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		primary_rev = execute_query(sql,val)

		sql = "SELECT H1.EmailID,IFNULL(H1.UserName,'') FROM HomeTable H1, ComponentReview C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
		val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		primary_des = execute_query(sql,val)

		for j in primary_rev:
			email_list.append(j[0])
			
		for j in primary_des:
			email_list.append(j[0])

		font_color = ''
		if q[3] == 1:	# signed-off
			font_color = 'green'
		elif q[3] == 2:	# ongoing
			font_color = 'orange'
		elif q[3] == 3:	# yet to kickstart
			font_color = 'red'

		if primary_des != ():
			status_interface += '<tr><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">'+q[1] + '</td><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">' + q[2] + '</td><td style="border-bottom: 1px solid #ddd;">' + primary_des[0][1] + '</td></tr>'
		else:
			status_interface += '<tr><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">'+q[1] + '</td><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">' + q[2] + '</td><td style="border-bottom: 1px solid #ddd;">-</td></tr>'

		status = status + q[1] + "&nbsp &nbsp &nbsp" + q[2] + "<br>"	


	status_interface += "</table>"

	message += status_interface

	message += '<br><br>Thanks,<br>ERAM.'

	catlead_sec_mail = []
	sql="SELECT DISTINCT a.CategoryName,b.EmailID,C3.CategoryLeadWWID1 from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2,BoardReviewDesigner B1 WHERE cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PLatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID AND C2.ComponentID = B1.ComponentID AND B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s ORDER BY cr.ComponentID"
	val=(sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],boardid,"yes")
	catlead=execute_query(sql,val)
	if(catlead != ()):
		for i in catlead:
			email_list.append(i[1])

			if i[2] is not None:
				if i[2] != []:
					cat_sec_wwid_list = i[2][1:-1].split(', ')

					for j in range(0,len(cat_sec_wwid_list)):
						like_user_wwid = '%' + str(cat_sec_wwid_list[j]) + '%'
						if cat_sec_wwid_list[j] not in ['99999999',99999999]:
							sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
							val=(like_user_wwid,)
							catlead_sec_mail_rs = execute_query(sql,val)

							if catlead_sec_mail_rs != ():
								email_list.append(catlead_sec_mail_rs[0][0])
	
	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val=('yes',)
	admin_list = execute_query(sql,val)				

	for j in admin_list:
		email_list.append(j[0])



	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	rid = execute_query(sql,val)[0][0]

	timeline = ""
	if(rid == 1):
		timeline = "Core/Placement layout review Rev0p6"
	else:
		timeline = "Full layout review Rev1p0"

			
	d1 = str(cal[0][1])	

	try:
		eids = []
		if(usernames):
			sql = "select EmailID from HomeTable where WWID in %s"
			val = (usernames,)
			eids = execute_query(sql, val)

		for i in range(len(eids)):
			if(eids[i][0]):
				email_list.append(eids[i][0])	
	except:
		pass

	email_list = sorted(set(email_list), reverse=True)
	for i in email_list:
		send_mail(i,subject,message,email_list)

	#return redirect('/design_summary_data_page?boardid='+str(boardid))
	return redirect(url_for("design_summary_data_page",boardid=str(boardid),_external=True))

@app.route("/title_page",methods = ['POST', 'GET'])
def title_page():
	boardid = request.form.get("boardid")
	wwid = session.get('wwid')

	sql = "SELECT BoardName, ClosureComment FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	result = execute_query(sql,val)
	boardname = result[0][0]
	closure_comment = result[0][1]

	sql = "SELECT ProposedEndDate FROM DesignCalendar WHERE BoardID =%s"
	val = (boardid,)
	enddate = execute_query(sql,val)
	EndDate=[]
	EndDate.append(enddate[0][0])

	if enddate[0][0]:
		ww_end_date = ' (WW' + str(get_isocalendar(enddate[0][0])[1]) + '.' + str(get_isocalendar(enddate[0][0])[2]) + ')'
	else:
		ww_end_date = ''

	EndDate.append(ww_end_date)


	sql = "SELECT a.BoardName,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,c.Username,DesignManagerWWID,d.Username,e.Username,core,IFNULL(f.RequestID,'') FROM BoardDetails a NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) LEFT JOIN HomeTable c ON a.DesignLeadWWID = c.WWID LEFT JOIN HomeTable d ON a.CADLeadWWID = d.WWID LEFT JOIN HomeTable e ON a.PIFLeadWWID = e.WWID LEFT JOIN RequestMap f ON a.BoardID=f.BoardID WHERE a.BoardID = %s"
	val = (boardid,)
	boarddeets = execute_query(sql,val)
	boarddeets_list = []
	for i in range(len(boarddeets)):
		role = boarddeets[len(boarddeets) - i - 1]
		boarddeets_list.append(role)

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)

	ww_date = []
	if cal[0][3]:
		cal_ww_start_date = ' (WW' + str(get_isocalendar(cal[0][3])[1]) + '.' + str(get_isocalendar(cal[0][3])[2]) + ')'
	else:
		cal_ww_start_date = ''
	ww_date.append(cal_ww_start_date)

	if cal[0][4]:
		cal_ww_end_date = ' (WW' + str(get_isocalendar(cal[0][4])[1]) + '.' + str(get_isocalendar(cal[0][4])[2]) + ')'
	else:
		cal_ww_end_date = ''
	ww_date.append(cal_ww_end_date)

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	sql = "SELECT DISTINCT B1.BoardID,B1.BoardID,B1.ComponentID,B1.PDG,B1.CommentDesigner,B1.PDG_Electrical,C1.ComponentName,S2.ScheduleTypeName,H1.UserName,H3.UserName,B1.CommentElectrical,H4.UserName,B1.CommentSignOffInterface,H5.UserName,B1.IsPdgElectricalSubmitted FROM BoardReviewDesigner B1, ComponentType C1, ScheduleTableComponent S1, ScheduleStatusType S2, ComponentReview C2, HomeTable H1, HomeTable H2, HomeTable H3, HomeTable H4, HomeTable H5 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID =%s AND B1.ComponentID = C2.ComponentID AND C2.PrimaryWWID = H1.WWID AND B1.CommentDesignUpdatedBy = H3.WWID AND B1.CommntElectricalUpdatedBy = H4.WWID AND B1.CommentSignOffInterfaceUpdateBy = H5.WWID ORDER BY FIELD(S1.ScheduleStatusID, 3, 6, 2, 1, 4, 5, 7), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
	val = (boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	comp = execute_query(sql,val)

	status = ""

	sql = "SELECT * FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	r = execute_query(sql,val)

	if(len(r)==0):
		status = "Yet to kickoff"
	else:
		sql = "SELECT S1.ScheduleTypeName FROM ScheduleStatusType S1, DesignCalendar D1, ScheduleTable S2 WHERE D1.BoardID = S2.BoardID AND S2.ScheduleStatusID = S1.ScheduleID AND D1.BoardID = %s "
		val = (boardid,)
		status = execute_query(sql,val)[0][0]	

	access_admin = 'no'	
	

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	if(has_admin_access == 'yes'):
		access_admin = 'yes'

	sql = "SELECT ht.UserName FROM BoardDetailsRequest br ,RequestMap rm, HomeTable ht WHERE rm.RequestID = br.RequestID AND br.WWID = ht.WWID AND rm.BoardID = %s"
	val = (boardid,)
	design_raised_by = ''

	try:
		design_raised_by = execute_query(sql,val)[0][0]
	except:
		pass

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	return render("boarddetails.html",username=username,user_role_name=user_role_name,region_name=region_name,BoardID=boardid,design_raised_by=design_raised_by,Boardname=boardname,closure_comment=closure_comment,boarddeets_list=boarddeets_list,cal = cal,comp = comp,Status = status,access_admin = access_admin,EndDate=EndDate,ww_date=ww_date)



#Method to implement Single Sign On.
@app.route('/sso')
def sso():
	print("sso login")
	user_token = request.args.get('token')
	token = Token(API_BASE_URL, 'v1')
	windows_auth = WindowsAuth(API_BASE_URL, 'v1')
	result = token.get_token(sys_account, sys_pwd)
	access_token = result.access_token
	expires_in = result.expires_in
	user_data = windows_auth.get_user_data(user_token, access_token)

	global email_id
	email_id = user_data['emails'][0]['value']

	#session.permanent = True

	session['username'] = user_data['displayName']
	session['wwid']= user_data['id']
	session['sso_loggedin_wwid']= user_data['id']
	session['email'] = user_data['emails'][0]['value']
	session['is_admin'] = False

	session['is_inactive_user'] = False
	session['inactive_user_msg'] = ""

	sql="SELECT a.IsActive,b.RoleName FROM HomeTable a LEFT JOIN RoleTable b ON a.RoleID=b.RoleID WHERE WWID=%s"
	val = (user_data['id'],)
	active_user_rs=execute_query(sql,val)

	if active_user_rs != ():
		if active_user_rs[0][0] == 0:
			session['is_inactive_user'] = True
			session['inactive_user_msg'] = 'Your <b>'+str(active_user_rs[0][1])+'</b> role access is being revoked. <br><br>Please <a href="/request_user_access">click here</a> to get access.'

	temp = []
	sql="SELECT DISTINCT a.EmailID FROM HomeTable a WHERE a.IsActive = %s"
	val = (0,)
	active_users=execute_query(sql,val)

	for i in range(0,len(active_users)):
		temp.append(active_users[i][0])

	global inactive_email_ids

	inactive_email_ids = copy.deepcopy(temp)

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (user_data['id'],)
	has_admin_access = execute_query(sql,val)

	if has_admin_access != ():
		if(has_admin_access[0][0] == 'yes'):
			session['is_admin'] = True


	sql="SELECT a.RoleID,b.RoleName FROM HomeTable a LEFT JOIN RoleTable b ON a.RoleID=b.RoleID WHERE WWID=%s "
	val = (user_data['id'],)
	role_rs=execute_query(sql,val)

	if role_rs != ():
		session['user_role_name'] = role_rs[0][1]
	else:
		session['user_role_name'] = ''

	if is_prod:
		session['region_name'] = ""
	else:
		session['region_name'] = copy.deepcopy(DB_NAME)

	if session['target_page'] == 'feedbacks':
		return redirect(url_for('feedbacks', _external=True))

	elif session['target_page'] == 'design_files':
		return redirect(url_for('design_files', _external=True))

	elif session['target_page'] == 'design_summary':
		return redirect(url_for('design_summary', _external=True))

	elif session['target_page'] == 'electrical_owners':
		return redirect(url_for('electrical_owners', _external=True))

	elif session['target_page'] == 'design_owners':
		return redirect(url_for('design_owners', _external=True))

	elif session['target_page'] == 'role_change':
		return redirect(url_for('role_change', _external=True))

	elif session['target_page'] == 'data_mining':
		return redirect(url_for('data_mining', _external=True))

	elif session['target_page'] == 'request_design':
		return redirect(url_for('request_design', _external=True))

	elif session['target_page'] == 'request_review':
		return redirect(url_for('request_review', _external=True))

	elif session['target_page'] == 'interface_add_update':
		return redirect(url_for('interface_add_update', _external=True))

	elif session['target_page'] == 'review_request':
		return redirect(url_for('review_request', _external=True))

	elif session['target_page'] == 'review_design':
		return redirect(url_for('review_design', _external=True))

	elif session['target_page'] == 'request_history':
		return redirect(url_for('request_history', _external=True))

	elif session['target_page'] == 'adm_que':
		return redirect(url_for('adm_que', _external=True))

	elif session['target_page'] == 'logs':
		return redirect(url_for('logs', _external=True))
	
	else:
		return redirect(url_for('index', _external=True))

@app.route("/request_user_access",methods = ['POST', 'GET'])
def request_user_access():
	sql = "SELECT RoleName from RoleTable WHERE RoleID <> 2 AND RoleID <> 11 "
	roles = execute_query_sql(sql)
	roles_list=[]
	for i in roles:
		role=i[0]
		roles_list.append(role)
	return render('request_access.html',role=roles_list,message="")

#Method called when the request access page is loaded. It gets the details and stores it in the DB.
@app.route('/request_access',methods = ['POST', 'GET'])
def request_access():
	username=session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	wwid_form=session.get('wwid')
	role = request.form.get('role')
	reason=request.form.get('reason')
	email = session.get('email')
	admin_access='no'
	current_date = datetime.datetime.now(tz).date()

	if(username == "" or username == None):
		return render('error.html',error='username',username=username,user_role_name=user_role_name,region_name=region_name)

	if(wwid_form == "" or wwid_form == None):
		return render('error.html',error='wwid',username=username,user_role_name=user_role_name,region_name=region_name)

	if(role== "" or  role== None):
		return render('error.html',error='role',username=username,user_role_name=user_role_name,region_name=region_name)

	if(reason== "" or  reason== None):
		return render('error.html',error='reason',username=username,user_role_name=user_role_name,region_name=region_name)

	if(email== "" or  email== None):
		return render('error.html',error='email',username=username,user_role_name=user_role_name,region_name=region_name)

	# log table
	try:
		log_notes = 'User has raised access request for ERAM tool.'
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
		val = ('Access Request',0,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")

	sql = "SELECT RoleID FROM RoleTable WHERE RoleName = %s"
	val=(role,)
	roleid = execute_query(sql,val)
	#roleid = role_query[0]

	#sql = "SELECT StateID FROM RequestState WHERE StateName = %s"
	#val = ('pending',)
	#stateid = execute_query(sql,val)
	#stateid = state_query[0]

	#sql = "SELECT * FROM RequestAccess WHERE WWID = %s AND RoleID = %s"
	#val = (wwid_form,roleid[0][0])
	#result_row = execute_query(sql,val)

	sql = "SELECT RequestID FROM RequestAccess WHERE WWID = %s AND StateID = %s"
	val = (wwid_form,1)
	requestid = execute_query(sql,val)

	if requestid != ():
		username = session.get('username')
		user_role_name = session.get('user_role_name')
		region_name = session.get('region_name')

		return render('pending.html',requestid=requestid[0][0],username=username,user_role_name=user_role_name,region_name=region_name)

	#if result_row:
	#	return render('error.html',error='Request. You have already requested for access. Do Not Re-submit or contact Admin.',username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "SELECT RequestID FROM RequestAccess WHERE WWID = %s ORDER BY RequestID DESC"
	val = (wwid_form,)
	request_rs = execute_query(sql,val)

	if request_rs != ():
		sql = "UPDATE RequestAccess SET RoleID = %s,Reason = %s,StateID = %s WHERE WWID = %s AND RequestID = %s"
		val = (roleid,reason,1,wwid_form,request_rs[0][0])
		insert = execute_query(sql,val)
	else:
		sql = "INSERT INTO RequestAccess (WWID,Username,RoleID,Reason,ExpiryDate,AdminAccess,StateID,EmailID) VALUES (%s,%s,%s,%s,%s,%s,%s,%s)"
		val = (wwid_form,username,roleid,reason,current_date,admin_access,1,email)
		insert = execute_query(sql,val)

	sql = "SELECT RequestID FROM RequestAccess WHERE WWID= %s"
	val=(wwid_form,)
	req_id = execute_query(sql,val)

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val=('yes',)
	admin_list = execute_query(sql,val)

	subject="Request for Review"
	message=''' Your action is required. <br>

	<b>Request ID: </b>''' + str(req_id[0][0]) + ''' <br>
	<b>Username: </b>''' + str(username) + '''<br>
	<b>WWID: </b>''' + str(wwid_form) + ''' <br>
	<b>Role: </b>''' + str(role) + ''' <br>
	<b>Reason: </b>''' + str(reason) + ''' <br>
	<b>Admin Access: </b>no <br>

	Please proceed to visit https://eram.apps1-fm-int.icloud.intel.com/ to approve or reject the request'''


	for admin in admin_list:
		send_mail(admin[0],subject,message)


	sql = "SELECT RequestID FROM RequestAccess WHERE WWID = %s"
	val=(wwid_form,)
	requestid_query = execute_query(sql,val)
	requestid = str(requestid_query[0][0])

	subject="ERAM Access"
	message = '''
	Your request id is:'''+ requestid+'''<br> 
	Please wait until your request is approved. '''

	send_mail(email,subject,message)

	#return redirect(url_for('index', _external=True))
	return render("submitted.html",username=username,user_role_name=user_role_name,region_name=region_name)

@app.route("/review_req.html")
def review_request():

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'review_request'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)


	wwid = session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)

	sql="SELECT RoleID FROM HomeTable WHERE WWID=%s "
	val = (wwid,)
	role=execute_query(sql,val)[0][0]

	if (role == 14):
		mgt_access=True
	else:
		mgt_access=False

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	if(has_admin_access[0][0] == 'yes'):
		sql = "SELECT RequestID,Username,WWID,RoleName,Reason,ExpiryDate,AdminAccess FROM RequestAccess NATURAL JOIN RoleTable WHERE StateID = 1 ORDER BY RequestID"
		result = execute_query_sql(sql)

		sql = "SELECT a.RequestID,d.UserName,a.WWID,b.RoleName,c.RoleName,a.Reason,a.Request_Time,e.StateName FROM RoleChangeRequest a LEFT JOIN RoleTable b ON a.CurrentRoleID = b.RoleID LEFT JOIN RoleTable c ON a.RequestedRoleID = c.RoleID LEFT JOIN HomeTable d ON d.WWID=a.WWID LEFT JOIN RequestState e ON a.StateID=e.StateID WHERE a.StateID = 1 ORDER BY RequestID"
		result_role = execute_query_sql(sql)

		return render("review_req.html",is_admin=is_admin,is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,username=username,user_role_name=user_role_name,region_name=region_name,request=result,admin='yes',mgt_access=mgt_access,request_role=result_role)

	else:
		return render("review_req.html",is_admin=is_admin,is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,username=username,user_role_name=user_role_name,region_name=region_name,request='',admin='no',mgt_access=mgt_access)


@app.route('/accept',methods = ['POST', 'GET'])
def accept():
	approveid = request.form.get('approveid')

	# log table
	try:
		log_notes = 'User has Accepted ERAM tool access request for Request ID: '+str(approveid)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
		val = ('Access Request',0,approveid,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")

	sql = "Update RequestAccess SET StateID = 2 WHERE RequestID = %s"
	val=(approveid,)
	accepted_request = execute_query(sql,val)

	sql = "SELECT WWID,Username,RoleID,AdminAccess,ExpiryDate,EmailID FROM RequestAccess WHERE RequestID = %s"
	val = (approveid,)

	approved_details = execute_query(sql,val)

	sql = "INSERT INTO HomeTable VALUES (%s,%s,%s,%s,%s,%s,%s) ON DUPLICATE KEY UPDATE RoleID = %s, IsActive = %s"
	val=(approved_details[0][0],approved_details[0][1],approved_details[0][2],approved_details[0][3],approved_details[0][4],approved_details[0][5],True,approved_details[0][2],1)
	execute_query(sql,val)

	sql = "SELECT EmailID FROM RequestAccess WHERE RequestID = %s"
	val = (approveid,)
	accepted_email_query = execute_query(sql,val)
	accepted_email = str(accepted_email_query[0][0])


	message = '''
		Hi,<br><br>Your request for access to ERAM has been accepted. <br><br>Please visit https://eram.apps1-fm-int.icloud.intel.com/ to proceed further. <br>'''
	subject="[Request ID:"+str(approveid)+"] Request for ERAM has been approved."

	email_list = []
	send_mail(accepted_email,subject,message,email_list)

	return redirect(url_for('review_request', _external=True))

@app.route('/reject',methods = ['POST', 'GET'])
def reject():
	rejectid = request.form.get('rejectid')
	rejectreason=request.form.get("reject_reason")
	user_wwid = session.get('wwid')
	username = session.get('username')

	# log table
	try:
		log_notes = 'User has Rejected Eram tool access request for Request ID: '+str(rejectid)+'<br>Reason: '+str(rejectreason)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
		val = ('Access Request',0,rejectid,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")
	

	sql = "SELECT EmailID FROM RequestAccess WHERE RequestID = %s"
	val = (rejectid,)
	rejected_email_query = execute_query(sql,val)
	rejected_email = str(rejected_email_query[0][0])


	sql = "DELETE FROM RequestAccess WHERE RequestID = %s"
	val=(rejectid,)
	rejected_request = execute_query(sql,val)



	message = ''' 
		Your request to access ERAM has been rejected. Please provide a proper justification or select a different role <br>
		Reason for rejection : ''' +rejectreason 
	subject = "[Request ID:"+str(rejectid)+"] Request to access ERAM has been rejected"
	email_list = []
	send_mail(rejected_email,subject,message,email_list)

	return redirect(url_for('review_request', _external=True))

@app.route('/design_electrical_owners', methods=['POST', 'GET'])
def design_electrical_owners():

	wwid =  session.get('wwid')
	username =  session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	boardid = request.form.get("boardid")

	data = {}

	sql="SELECT BoardName,ComponentName,CategoryName,IF(e.UserName='N.A','',e.UserName),SecondaryCategoryLead,IF(f.UserName='N.A','',f.UserName),SecondaryElectricalOwner FROM DesignElectricalOwners a NATURAL JOIN (BoardDetails b,ComponentType d) LEFT JOIN CategoryType c ON c.CategoryID = a.CategoryID LEFT JOIN HomeTable e ON e.WWID = a.PrimaryCategoryLead LEFT JOIN HomeTable f ON f.WWID = a.PrimaryElectricalOwner LEFT JOIN BoardReviewDesigner g ON g.BoardID = a.BoardID AND g.ComponentID = a.ComponentID WHERE a.BoardID = %s AND g.BoardID IS NOT NULL ORDER BY c.CategoryName,ComponentName ASC"
	val = (boardid,)
	result=execute_query(sql,val)

	board_name = ''

	if result !=():

		board_name = result[0][0]

		for i in range(len(result)):

			data[i] = []
			data[i].append(result[i][0])
			data[i].append(result[i][1])
			data[i].append(result[i][2])
			data[i].append(result[i][3])

			SecondaryCategoryLead = ''
			if(result[i][4] != None):
				rem = result[i][4][1:-1]
				if(rem != None or  rem != "" ):
					spl = rem.split(",")
					if (spl != ['']):
						for j in range(0,len(spl)):
							lead1_wwid = (spl[j])
							if len(lead1_wwid) >= 8:
								if str(lead1_wwid) != str(99999999):
									sql = "select UserName from HomeTable where WWID = %s"
									val=(str(lead1_wwid),)
									name = execute_query(sql,val)
									if SecondaryCategoryLead == '':
										SecondaryCategoryLead = name[0][0]
									else:
										SecondaryCategoryLead = SecondaryCategoryLead + ';  ' + name[0][0]
			data[i].append(SecondaryCategoryLead)

			data[i].append(result[i][5])

			SecondaryElectricalOwner = ''
			if(result[i][6] != None):
				rem = result[i][6][1:-1]
				if(rem != None or  rem != "" ):
					spl = rem.split(",")
					if (spl != ['']):
						for j in range(0,len(spl)):
							lead1_wwid = (spl[j])
							if len(lead1_wwid) >= 8:
								if str(lead1_wwid) != str(99999999):
									sql = "select UserName from HomeTable where WWID = %s"
									val=(str(lead1_wwid),)
									name = execute_query(sql,val)
									if SecondaryElectricalOwner == '':
										SecondaryElectricalOwner = name[0][0]
									else:
										SecondaryElectricalOwner = SecondaryElectricalOwner + ';  ' + name[0][0]
			data[i].append(SecondaryElectricalOwner)			

	return render("design_electrical_owners.html",username=username,user_role_name=user_role_name,region_name=region_name,board_name=board_name,boardid=boardid,data=data)


@app.route('/design_design_owners', methods=['POST', 'GET'])
def design_design_owners():

	wwid =  session.get('wwid')
	username =  session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	boardid = request.form.get("boardid")

	data = {}

	sql="SELECT BoardName,ComponentName,CategoryName,IF(f.UserName='N.A','',f.UserName),SecondaryOwner FROM DesignDesignOwners a NATURAL JOIN (BoardDetails b,ComponentType d) LEFT JOIN CategoryType c ON c.CategoryID = a.CategoryID LEFT JOIN HomeTable f ON f.WWID = a.PrimaryOwner LEFT JOIN BoardReviewDesigner g ON g.BoardID = a.BoardID AND g.ComponentID = a.ComponentID WHERE a.BoardID = %s AND g.BoardID IS NOT NULL ORDER BY c.CategoryName,ComponentName ASC"
	val = (boardid,)
	result=execute_query(sql,val)

	board_name = ''

	if result !=():

		board_name = result[0][0]

		for i in range(len(result)):

			data[i] = []
			data[i].append(result[i][0])
			data[i].append(result[i][1])
			data[i].append(result[i][2])
			data[i].append(result[i][3])

			SecondaryOwner = ''
			if(result[i][4] != None):
				rem = result[i][4][1:-1]
				if(rem != None or  rem != "" ):
					spl = rem.split(",")
					if (spl != ['']):
						for j in range(0,len(spl)):
							lead1_wwid = (spl[j])
							if len(lead1_wwid) >= 8:
								if str(lead1_wwid) != str(99999999):
									sql = "select UserName from HomeTable where WWID = %s"
									val=(str(lead1_wwid),)
									name = execute_query(sql,val)
									if SecondaryOwner == '':
										SecondaryOwner = name[0][0]
									else:
										SecondaryOwner = SecondaryOwner + ';  ' + name[0][0]
			data[i].append(SecondaryOwner)			

	return render("design_design_owners.html",username=username,user_role_name=user_role_name,region_name=region_name,board_name=board_name,boardid=boardid,data=data)

@app.route('/design_electrical_owners_sync', methods=['POST', 'GET'])
def design_electrical_owners_sync():

	sql="SELECT BoardID,PlatformID,SKUID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = 70 ORDER BY BoardID ASC"
	bdetails=execute_query_sql(sql)

	if bdetails == ():
		return "No data from bdetails."

	for row in bdetails:

		sql="SELECT ComponentID,CategoryID,PlatformID,SKUID,MemTypeID,DesignTypeID,PrimaryWWID,SecondaryWWID FROM ComponentReview WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s ORDER BY ComponentID ASC"
		val = (row[1],row[2],row[3],row[4])
		compdetails = execute_query(sql,val)

		if compdetails != ():

			for comp_row in compdetails:

				sql="SELECT CategoryLeadWWID,CategoryLeadWWID1 FROM CategoryLeadTable WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s AND CategoryID = %s"
				val = (row[1],row[2],row[3],row[4],comp_row[1])
				categorydetails = execute_query(sql,val)

				prim_cad_lead = 99999999
				sec_cad_lead = '[]'

				if categorydetails != ():
					prim_cad_lead = categorydetails[0][0]
					sec_cad_lead = categorydetails[0][1]

				sql = "INSERT INTO DesignElectricalOwners VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
				val = (row[0],comp_row[0],comp_row[1],comp_row[2],comp_row[3],comp_row[4],comp_row[5],prim_cad_lead,sec_cad_lead,comp_row[6],comp_row[7])
				execute_query(sql, val)

	return "done."


@app.route('/design_design_owners_sync', methods=['POST', 'GET'])
def design_design_owners_sync():

	sql="SELECT BoardID,PlatformID,SKUID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = 70 ORDER BY BoardID ASC"
	bdetails=execute_query_sql(sql)

	if bdetails == ():
		return "No data from bdetails."

	for row in bdetails:

		sql="SELECT ComponentID,CategoryID,PlatformID,SKUID,MemTypeID,DesignTypeID,PrimaryWWID,SecondaryWWID FROM ComponentDesign WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s ORDER BY ComponentID ASC"
		val = (row[1],row[2],row[3],row[4])
		compdetails = execute_query(sql,val)

		if compdetails != ():

			for comp_row in compdetails:

				sql = "INSERT INTO DesignDesignOwners VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s)"
				val = (row[0],comp_row[0],comp_row[1],comp_row[2],comp_row[3],comp_row[4],comp_row[5],comp_row[6],comp_row[7])
				execute_query(sql, val)

	return "done."

def sync_design_electrical_owners():

	# to sync and freeze Design Electrical Owners
	sql="SELECT DISTINCT a.BoardID,a.PlatformID,a.SKUID,a.MemTypeID,a.DesignTypeID FROM BoardDetails a, ScheduleTable b WHERE a.BoardID = b.BoardID AND b.ScheduleStatusID IN (2,3,6) ORDER BY a.BoardID"
	bdetails=execute_query_sql(sql)

	for row in bdetails:

		sql="SELECT ComponentID,CategoryID,PlatformID,SKUID,MemTypeID,DesignTypeID,PrimaryWWID,SecondaryWWID FROM ComponentReview WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s ORDER BY ComponentID ASC"
		val = (row[1],row[2],row[3],row[4])
		compdetails = execute_query(sql,val)

		if compdetails != ():

			for comp_row in compdetails:

				sql="SELECT CategoryLeadWWID,CategoryLeadWWID1 FROM CategoryLeadTable WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s AND CategoryID = %s"
				val = (row[1],row[2],row[3],row[4],comp_row[1])
				categorydetails = execute_query(sql,val)

				prim_cad_lead = 99999999
				sec_cad_lead = '[]'

				if categorydetails != ():
					prim_cad_lead = categorydetails[0][0]
					sec_cad_lead = categorydetails[0][1]

				sql = "INSERT INTO DesignElectricalOwners VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s) ON DUPLICATE KEY UPDATE PrimaryCategoryLead = %s, SecondaryCategoryLead = %s, PrimaryElectricalOwner = %s, SecondaryElectricalOwner = %s"
				val = (row[0],comp_row[0],comp_row[1],comp_row[2],comp_row[3],comp_row[4],comp_row[5],prim_cad_lead,sec_cad_lead,comp_row[6],comp_row[7],prim_cad_lead,sec_cad_lead,comp_row[6],comp_row[7])
				execute_query(sql, val)

	return True

@app.route('/interface_add_update.html', methods=['POST', 'GET'])
def interface_add_update():

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'interface_add_update'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	name =  session.get('username')
	wwid =  session.get('wwid')

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "Select C1.ComponentID,C1.ComponentName,C1.ComponentDescription,C1.CategoryID,C1.LitePiComponentName,C2.CategoryName from ComponentType C1,CategoryType C2 WHERE C1.CategoryID = C2.CategoryID order by C1.CategoryID ASC"
	comps = execute_query_sql(sql)

	sql = "select CategoryName from CategoryType order by CategoryID asc "
	category = execute_query_sql(sql)
	cat_list = []
	for i in category:
		cat_list.append(i[0])

	sql="select ComponentName from ComponentType order by ComponentID asc "

	idlist = execute_query_sql(sql)
	compid=[]
	for i in idlist:
		compid.append(i[0])

	sql="select RoleID from HomeTable where WWID=%s "
	val = (wwid,)
	role=execute_query(sql,val)[0][0]
	if (role == 14):
		mgt_access=True
	else:
		mgt_access=False

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	sql="select AdminAccess,RoleID from HomeTable where WWID=%s"
	val=(wwid,)
	hasaccess=execute_query(sql,val)

	if hasaccess[0][0]=="yes":
		admin=True
	else:
		admin=False

	return render("interface_add_update.html",is_admin=is_admin,is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,category=cat_list,compid=compid,mgt_access=mgt_access,admin=admin,name=name,comps=comps,username=username,user_role_name=user_role_name,region_name=region_name)

@app.route('/add_component_type',methods = ['POST', 'GET'])
def add_component_type():
	compname = request.form.get('componentname')
	compdesc = request.form.get('componentdesc')
	componentcategory=request.form.get('componentcategory')

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	sql="select CategoryID from CategoryType where CategoryName=%s"
	val=(componentcategory,)
	catid=execute_query(sql,val)

	sql="select * from ComponentType where ComponentName=%s"
	val=(compname,)
	comp_name=execute_query(sql,val)

	if(comp_name!=()):
		return render('error_custom.html',error='Component Name is already available.',username=username,user_role_name=user_role_name,region_name=region_name)

	# log table
	try:
		log_notes = 'User has added new Component details:'
		log_notes += '<br>Component Name: '+str(compname)+'<br>Category: '+str(componentcategory)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Component details',0,0,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	lite_pi_component_name = get_modified_comp_name_for_litepi(comp_name=copy.deepcopy(compname))

	sql="INSERT INTO ComponentType VALUES (%s,%s,%s,%s,%s)"
	val=(0,compname,compdesc,catid[0][0],lite_pi_component_name)
	execute_query(sql, val)

	return redirect(url_for('interface_add_update', _external=True))

def get_modified_comp_name_for_litepi(comp_name=""):

	litep_comp_name = copy.deepcopy(comp_name.strip())

	litep_comp_name = litep_comp_name.replace(" ","_")
	litep_comp_name = litep_comp_name.replace("-","_")
	litep_comp_name = litep_comp_name.replace("'","_")
	litep_comp_name = litep_comp_name.replace('"',"_")
	litep_comp_name = litep_comp_name.replace("*","_")
	litep_comp_name = litep_comp_name.replace(":","_")
	litep_comp_name = litep_comp_name.replace(";","_")
	litep_comp_name = litep_comp_name.replace("#","_")
	litep_comp_name = litep_comp_name.replace("@","_")
	litep_comp_name = litep_comp_name.replace("!","_")
	litep_comp_name = litep_comp_name.replace("`","_")
	litep_comp_name = litep_comp_name.replace("~","_")
	litep_comp_name = litep_comp_name.replace("%","_")
	litep_comp_name = litep_comp_name.replace("\\","_")
	litep_comp_name = litep_comp_name.replace(".","p")
	litep_comp_name = litep_comp_name.replace("+","plus")
	litep_comp_name = litep_comp_name.replace("&","and")
	litep_comp_name = litep_comp_name.replace(",","_")
	litep_comp_name = litep_comp_name.replace("/","_or_")
	litep_comp_name = litep_comp_name.replace("(","_")
	litep_comp_name = litep_comp_name.replace(")","")
	litep_comp_name = litep_comp_name.replace("__","_")
	litep_comp_name = litep_comp_name.replace("__","_")
	litep_comp_name = litep_comp_name.replace("__","_")
	litep_comp_name = litep_comp_name.replace("__","_")
	litep_comp_name = litep_comp_name.replace("__","_")

	return litep_comp_name

@app.route('/modify_component_name_for_litepi',methods = ['POST', 'GET'])
def modify_component_name_for_litepi():

	sql = "SELECT ComponentID,ComponentName FROM ComponentType ORDER BY ComponentID ASC"
	rs = execute_query_sql(sql)

	for row in rs:
		litep_comp_name = get_modified_comp_name_for_litepi(comp_name=row[1])

		sql = "UPDATE ComponentType SET LitePiComponentName = %s WHERE ComponentID = %s"
		val = (litep_comp_name,row[0])
		execute_query(sql,val)

	return "Done."

@app.route('/modify_component_type',methods = ['POST', 'GET'])
def modify_component_type():
	compid = request.form.get('componenttype_id')
	newcompname=request.form.get('componenttypename')
	componentcategory=request.form.get('componentcategory')
	compid = compid.replace("&amp;","&")
	newcompname = newcompname.replace("&amp;","&")
	if(componentcategory):
		sql="select CategoryID from CategoryType where CategoryName=%s"
		val=(componentcategory,)
		catid=execute_query(sql,val)

	sql="select ComponentID from ComponentType where ComponentName=%s"
	val=(compid,)
	compoid=execute_query(sql,val)

	if(componentcategory):
		# log table
		try:
			log_notes = 'User has Modified Component details. '
			log_notes += '<br>Component Name: '+str(newcompname)+'<br>Category: '+str(componentcategory)
			log_wwid = session.get('wwid')
			t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
			sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
			val = ('Component details',0,0,compoid[0][0],log_wwid,t,log_notes)
			execute_query(sql,val)
		except:
			if is_logging:
				logging.exception('')
			print("log error.")		
				
		sql="UPDATE ComponentType SET ComponentName=%s,CategoryID=%s where ComponentID=%s "
		val=(newcompname,catid[0][0],compoid[0][0])
		execute_query(sql, val)
		return redirect(url_for('interface_add_update', _external=True))


@app.route('/del_component_type',methods = ['POST', 'GET'])
def del_component_type():
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	compid = request.form.get('componenttype_id')
	compid = compid.replace("&amp;","&")
	sql="select ComponentID from ComponentType where ComponentName=%s"
	val=(compid,)
	compoid=execute_query(sql,val)
	sql="select ComponentID from BoardReviewDesigner"
	usedcomp=execute_query_sql(sql)
	usedcomps=[]
	for i in usedcomp:
		if i[0] not in usedcomps:
			usedcomps.append(i[0])
	if (int(compoid[0][0]) in usedcomps):
		return render('error.html',error="Cannot delete. This component is already assigned to a board",username=username,user_role_name=user_role_name,region_name=region_name)

	# log table
	try:
		log_notes = 'User has Deteled Component. '
		log_notes += '<br>Component Name: '+str(compid)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Component details',0,0,compoid[0][0],log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")		
			
	sql="DELETE FROM ComponentDesign WHERE ComponentID=%s"
	val=(compoid[0][0],)
	execute_query(sql, val)

	sql="DELETE FROM ComponentType WHERE ComponentID=%s "
	val=(compoid[0][0],)
	execute_query(sql, val)
	return redirect(url_for('interface_add_update', _external=True))

@app.route('/update_calendar',methods = ['POST', 'GET'])
def update_calendar():
	boardid=request.form.get('boardid')

	start=request.form.get('start')
	end=request.form.get('end')
	boardstate=request.form.get('boardstate')

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	# log table
	try:
		log_notes = 'User has Updated Calendar for Board ID: '+str(boardid)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Calendar',boardid,0,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")

	# to update last updated time
	sql="UPDATE BoardDetails set UpdatedOn = %s where BoardID=%s"
	val=(t,boardid)
	execute_query(sql,val)

	sql = "SELECT ProposedStartDate,ProposedEndDate,BoardState FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	rs_bfr_cal = execute_query(sql,val)

	if(start == "" or start == None):
		sql = "INSERT INTO DesignCalendar VALUES(%s,%s,%s,0000-00-00,0000-00-00,%s,null) ON DUPLICATE KEY UPDATE BoardState=%s"
		val = (boardid, start, end, boardstate,boardstate,)
		execute_query(sql, val)
		return redirect(url_for('index', _external=True))
	elif (end == "" or end == None):
		sql = "INSERT INTO DesignCalendar VALUES(%s,%s,%s,0000-00-00,0000-00-00,%s,null) ON DUPLICATE KEY UPDATE BoardState=%s"
		val = (boardid, start, end, boardstate,boardstate,)
		execute_query(sql, val)
		return redirect(url_for('index', _external=True))
	elif (boardstate == "" or boardstate == None ):
		sql = "INSERT INTO DesignCalendar VALUES(%s,%s,%s,0000-00-00,0000-00-00,%s,null) ON DUPLICATE KEY UPDATE ProposedStartDate =%s,ProposedEndDate = %s"
		val = (boardid, start, end, "z",start,end,)
		execute_query(sql, val)
		return redirect(url_for('index', _external=True))

	else:
		sql = "INSERT INTO DesignCalendar VALUES(%s,%s,%s,0000-00-00,0000-00-00,%s,null) ON DUPLICATE KEY UPDATE ProposedStartDate =%s,ProposedEndDate = %s, BoardState=%s"
		val=(boardid,start,end,boardstate,start,end,boardstate,)
		execute_query(sql,val)

		sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
		val = (boardid,)
		boardname = execute_query(sql,val)[0][0]

		email_list = []
		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
		val = (boardid,)
		designlist = execute_query(query,val)
		designlead_list = []
		for i in range(len(designlist)):
			eid = designlist[0][1]
			email_list.append(eid)


		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
		val = (boardid,)
		cadlist = execute_query(query,val)
		cadlead_list = []
		for i in range(len(cadlist)):
			eid = cadlist[0][1]
			email_list.append(eid)


		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.PIFLeadWWID and b.BoardID AND b.BoardID = %s)"
		piflist = execute_query(query,val)
		piflead_list = []
		for i in range(len(piflist)):
			eid = piflist[0][1]
			email_list.append(eid)

		# all pif leads
		email_list += get_pif_leads_email_id_by_board_id(boardid=boardid)

		start_date_highlight = False
		end_date_highliht = False
		state_highlight = False

		if rs_bfr_cal != ():
			sql = "SELECT ProposedStartDate,ProposedEndDate,BoardState FROM DesignCalendar WHERE BoardID = %s"
			val = (boardid,)
			rs_after_cal = execute_query(sql,val)

			if rs_after_cal != ():

				if rs_bfr_cal[0][0] != rs_after_cal[0][0]:
					start_date_highlight = True

				if rs_bfr_cal[0][1] != rs_after_cal[0][1]:
					end_date_highliht = True

				if rs_bfr_cal[0][2] != rs_after_cal[0][2]:
					state_highlight = True


		sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
		val=('yes',)
		admin_list = execute_query(sql,val)				

		for j in admin_list:
			email_list.append(j[0])


		subject =  "[ID: "+str(boardid)+"] Details Updated"
		message = '''Please find the updated details for <b>'''+boardname+'''</b><br>'''

		if start_date_highlight:
			message += '''<br><u>Start Date: '''+ get_work_week_str_fmt(start) +'''</u>'''
		else:
			message += '''<br>Start Date: '''+ get_work_week_str_fmt(start)

		if end_date_highliht:
			message += '''<br><u>End Date: ''' + get_work_week_str_fmt(end) + ''' </u>'''
		else:
			message += '''<br>End Date: ''' + get_work_week_str_fmt(end)

		if state_highlight:
			message += '''<br><u>Design State: ''' + boardstate + '''</u>'''
		else:
			message += '''<br>Design State: ''' + boardstate

		message += '''<br><br>Regards,<br>ERAM.'''

		if boardstate not in ('Design Signed-off','Design Not signed-off','Design Review In-Progress'):

			email_list = sorted(set(email_list), reverse=True)
			for i in email_list:
				send_mail(i,subject,message,email_list)

		return redirect(url_for('index', _external=True))

@app.route('/delete_board',methods = ['POST', 'GET'])
def del_board():
	delboard=request.form.get('boardid')

	# log table
	try:
		log_notes = 'User has Deleted Board ID: '+str(delboard)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Calendar',delboard,0,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")

	sql="select BoardID from BoardDetails where BoardName=%s"
	val=(delboard,)
	boardid=	execute_query(sql,val)

	sql="delete from DesignCalendar where BoardID=%s"
	val=(boardid[0][0],)
	execute_query(sql, val)

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid[0][0],)
	boardname = execute_query(sql,val)[0][0]

	email_list = []
	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
	val = (boardid[0][0],)
	designlist = execute_query(query,val)
	designlead_list = []
	for i in range(len(designlist)):
		eid = designlist[0][1]
		email_list.append(eid)


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
	val = (boardid[0][0],)
	cadlist = execute_query(query,val)
	cadlead_list = []
	for i in range(len(cadlist)):
		eid = cadlist[0][1]
		email_list.append(eid)


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.PIFLeadWWID and b.BoardID AND b.BoardID = %s)"
	piflist = execute_query(query,val)
	piflead_list = []
	for i in range(len(piflist)):
		eid = piflist[0][1]
		email_list.append(eid)

	# all pif leads
	email_list += get_pif_leads_email_id_by_board_id(boardid=boardid[0][0])

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val=('yes',)
	admin_list = execute_query(sql,val)				

	for j in admin_list:
		email_list.append(j[0])


	subject =  "[ID:"+str(boardid[0][0])+"] has been dropped" 	
	message = ''' The design '''+boardname+''' has been dropped <br>

		Thanks,<br>
		ERAM '''

	email_list = sorted(set(email_list), reverse=True)
	for i in email_list:
		send_mail(i,subject,message,email_list)	

	return redirect(url_for('index', _external=True))

@app.route('/design_req.html',methods = ['POST', 'GET'])   	##method for designer to raise board request
def request_design():

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'request_design'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	query = 'SELECT DesignTypeName FROM DesignType where DesignStatus="Approved"'
	items = execute_query_sql(query)
	items_list = []
	for i in items:
		role = i[0]
		items_list.append(role)

	query = "SELECT BoardTrackName FROM BoardTrack"
	track = execute_query_sql(query)
	track_list = []
	for i in track:
		role = i[0]
		track_list.append(role)

	#query = "SELECT DISTINCT core FROM BoardDetails ORDER BY core"
	query = "SELECT DISTINCT CoreName FROM Core WHERE CoreStatus='Approved' ORDER BY CoreName DESC"
	core = execute_query_sql(query)
	core_list = []
	for i in core:
		role = i[0]
		core_list.append(role)

	#if '682' not in core_list:
	#	core_list.append('682')

	query = "SELECT PlatformName FROM Platform WHERE PlatformStatus='Approved' ORDER BY FIELD(PlatformID,1,37,12,40,39,42,41,36,43,44,45,46,47,48,49,50)"
	platform = execute_query_sql(query)
	platform_list = []
	for i in platform:
		plat = i[0]
		platform_list.append(plat)

	query = "SELECT SKUName FROM SUK WHERE SKUStatus='Approved' ORDER BY FIELD(SKUID,28,29,24,8,9,27,25,10,30,31,32,33,34,35)"
	sku = execute_query_sql(query)
	sku_list = []
	for i in sku:
		p = i[0]
		sku_list.append(p)


	query = "SELECT MemTypeName FROM MemType where MemTypeStatus='Approved'"
	memtype = execute_query_sql(query)
	memtype_list = []
	for i in memtype:
		mem = i[0]
		memtype_list.append(mem)

	query = "SELECT Username FROM HomeTable WHERE IsActive = 1 AND RoleID IN (7,8) "
	cadlead = execute_query_sql(query)
	cadlead_list = []
	for i in range(len(cadlead)):
		user = cadlead[i]
		cadlead_list.append(user)

	query = "SELECT Username FROM HomeTable WHERE IsActive = 1 AND RoleID='3'"
	des_manager = execute_query_sql(query)
	desmanager_list = []
	for i in range(len(des_manager)):
		user = des_manager[i]
		desmanager_list.append(user)


	query = "SELECT Username FROM HomeTable WHERE IsActive = 1 AND RoleID='5'"
	deslead = execute_query_sql(query)
	deslead_list = []
	for i in range(len(deslead)):
		user = deslead[i]
		deslead_list.append(user)

	query = "SELECT Username FROM HomeTable WHERE IsActive = 1 AND RoleID IN (1,14)"
	piflead = execute_query_sql(query)
	piflead_list = []
	for i in range(len(piflead)):
		user = piflead[i]
		piflead_list.append(user)

	# Reference design
	design_status_list_ref = []
	design_list_ref = []
	#my_designs_id_ref = []

	temp_data_ref = []
	temp_data_ref = get_all_designs_feedbacks()

	#for i in range(0,len(temp_data_ref[1])):
	#	my_designs_id_ref.append(temp_data_ref[1][i][0])

	#design_status_list = get_status_list_sorted(data_list=temp_data[0])
	design_status_list_ref = get_order_status_list(list=temp_data_ref[0])

	design_list_ref = temp_data_ref[1]

	return render("design_req.html",design_status_list_ref=design_status_list_ref,design_list_ref=design_list_ref,username=username,user_role_name=user_role_name,region_name=region_name,items=items_list,core_list=core_list,tracks=track_list,deslead=deslead_list,cadlead=cadlead_list,piflead=piflead_list,platform=platform_list,sku=sku_list,memtype=memtype_list,des_manager=desmanager_list)


#On submitting design review request. 
@app.route('/submit', methods=['POST', 'GET'])
def request_form():
	name = session.get('username')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	user_wwid=session.get('wwid')
	core=request.form.get('core')
	core2 = core
	designtype = request.form.get('designtype')
	designtype2 = designtype
	timeline=request.form.get('timeline')
	timeline2 = timeline
	boardstate = request.form.get('type')
	boardstate2 = boardstate
	memorytype=request.form.get('memorytype')
	memorytype2 = memorytype
	sku=request.form.get('sku')
	sku2 = sku
	platform=request.form.get('platform')
	platform2 = platform
	boardtrack = request.form.get('track')
	boardtrack2 = boardtrack

	start = request.form.get('start')
	end = request.form.get('end')
	DesignLeadName = request.form.get('designlead')
	DesignManager = request.form.get('design_manager')
	CADlead = request.form.get('cadlead')
	PIFlead=request.form.get('piflead')
	comments = request.form.get('comments')
	combination_design = request.form.get('combination_design')
	design_request_IDs = request.form.get('design_request_IDs')

	ref_design_selection = request.form.get('ref_design_selection')
	new_ref_design_name = request.form.get('new_ref_design_name')

	if ref_design_selection == "old":
		boardid_ref = request.form.get('boardid_ref')
		ref_design_name = ""

		if boardid_ref == "":
			boardid_ref = 0

		sql = 'SELECT BoardName FROM BoardDetails WHERE BoardID = %s '
		val = (boardid_ref,)
		boardid_ref_rs = execute_query(sql, val)

		if boardid_ref_rs != ():
			ref_design_name = boardid_ref_rs[0][0]

	else:
		boardid_ref = 0
		ref_design_name = request.form.get('new_ref_design_name')
	
	if (designtype == "" or designtype == None):
		return render('error.html', error='designtype',username=username,user_role_name=user_role_name,region_name=region_name)

	if (timeline == "" or timeline == None):
		return render('error.html', error='timeline',username=username,user_role_name=user_role_name,region_name=region_name)

	if (boardtrack == "" or boardtrack == None):								###ADD DESIGN MAGER AND BOARD TYPE HERE ONCE THEY ARE CONNECTED TO DB
		return render('error.html', error='boardtrack',username=username,user_role_name=user_role_name,region_name=region_name)

	if (start == "" or start == None or valid_date(start)==False ):
		return render('error.html', error='startdate',username=username,user_role_name=user_role_name,region_name=region_name)

	if (end == "" or end == None or valid_date(end)==False):
		return render('error.html', error='enddate',username=username,user_role_name=user_role_name,region_name=region_name)

	if (DesignLeadName == "" or DesignLeadName == None):
		return render('error.html', error='date',username=username,user_role_name=user_role_name,region_name=region_name)

	if (PIFlead == "" or PIFlead == None):
		return render('error.html', error='PIFlead',username=username,user_role_name=user_role_name,region_name=region_name)

	if (boardtrack == "Yes" and (comments=="" or comments==None) ):								###ADD DESIGN MAGER AND BOARD TYPE HERE ONCE THEY ARE CONNECTED TO DB
		return render('error.html', error='Please add reason in comments for fastrack design',username=username,user_role_name=user_role_name,region_name=region_name)

	if(start>end):
		return render('error.html', error='start date greater than end date',username=username,user_role_name=user_role_name,region_name=region_name)


	email_list = []

	sql="select UserName from HomeTable where RoleID in (1,5,7,14,3,8)"
	validusername=execute_query_sql(sql)
	validuser=[]
	for i in validusername:
		validuser.append(i[0])

	if(((DesignLeadName in validuser)==False) or ((DesignManager in validuser)==False) or ((PIFlead in validuser)==False) or ((CADlead in validuser)==False)):
		return render('error_custom.html', error="The lead you entered is not valid. Please select from the dropdown and try again!",username=username,user_role_name=user_role_name,region_name=region_name)


	sql="SELECT DesignTypeName FROM DesignType "
	design_name = execute_query_sql(sql)
	design_name_list=[]
	for i in design_name:
		design_name_list.append(i[0])											##FOR DESIGN TYPE

	if ((designtype in design_name_list) == False):
		return render('error_custom.html', error="Invalid Design Type. Please select from the dropdown and try again!",username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "SELECT MemTypeName FROM MemType "
	memtype_name = execute_query_sql(sql)
	memtype_name_list = []
	for i in memtype_name:
		memtype_name_list.append(i[0])  ##FOR Memory TYPE

	if ((memorytype in memtype_name_list) == False):
		return render('error_custom.html', error="Invalid Memory Type. Please select from the dropdown and try again!",username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "SELECT SKUName FROM SUK "
	sku_name = execute_query_sql(sql)
	sku_name_list = []
	for i in sku_name:
		sku_name_list.append(i[0])  ##FOR SKU TYPE

	if ((sku in sku_name_list) == False):
		return render('error_custom.html', error="Invalid SKU. Please select from the dropdown and try again!",username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "SELECT PlatformName FROM Platform "
	platform_name = execute_query_sql(sql)
	platform_name_list = []
	for i in platform_name:
		platform_name_list.append(i[0])  ##FOR Memory TYPE

	if ((platform in platform_name_list) == False):
		return render('error_custom.html', error="Invalid Platform. Please select from the dropdown and try again!",username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "SELECT CoreName FROM Core "
	core_name = execute_query_sql(sql)
	core_name_list = []
	for i in core_name:
		core_name_list.append(i[0])  ##FOR Memory TYPE

	if ((core in core_name_list) == False):
		return render('error_custom.html', error="Invalid Core. Please select from the dropdown and try again!",username=username,user_role_name=user_role_name,region_name=region_name)


	sql = 'SELECT DesignTypeID FROM DesignType WHERE DesignTypeName = %s '
	val = (designtype,)
	designtype=execute_query(sql,val)[0][0]

	sql = ' SELECT MemTypeID FROM MemType WHERE MemTypeName = %s '
	val = (memorytype,)
	memorytype = execute_query(sql, val)[0][0]

	sql = ' SELECT SKUID FROM SUK WHERE SKUName = %s '
	val = (sku,)
	sku = execute_query(sql, val)[0][0]

	sql = ' SELECT PlatformID FROM Platform WHERE PlatformName = %s '
	val = (platform,)
	platform = execute_query(sql, val)[0][0]

	sql = "SELECT BoardStateID FROM BoardState WHERE BoardStateName='Pending'"
	boardstate = execute_query_sql(sql)[0][0]


	sql = "SELECT BoardTrackID FROM BoardTrack WHERE BoardTrackName= %s"
	val=(boardtrack,)
	boardtrack = execute_query(sql,val)[0][0]


	sql = "SELECT ReviewTimelineID FROM ReviewTimeline WHERE ReviewTimelineName=%s"
	val=(timeline,)
	timeline = execute_query(sql,val)[0][0]

	sql = "SELECT WWID FROM RequestAccess WHERE Username=%s"
	val=(DesignLeadName,)
	deslead = execute_query(sql,val)[0][0]

	sql = "SELECT WWID FROM RequestAccess WHERE Username=%s"
	val = (CADlead,)
	cadlead = execute_query(sql,val)[0][0]

	sql = "SELECT WWID FROM RequestAccess WHERE Username=%s"
	val = (PIFlead,)
	piflead = execute_query(sql,val)[0][0]

	# all pif leads
	#email_list += get_pif_leads_email_id_by_platform_id(platformid=platform)

	sql = "SELECT EmailID FROM RequestAccess WHERE WWID=%s"
	val = (user_wwid,)                                                   ###### CHANGE THIS TO session.get('wwid')
	user_email = execute_query(sql,val)


	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val = ('yes',)
	admin_email = execute_query(sql,val)

	com = comments

	sql = "INSERT INTO BoardDetailsRequest(DesignTypeID,BoardStateID,BoardTrackID,ReviewTimelineID,PlatformID,SKUID,MemTypeID,DesignLeadWWID,DesignManagerWWID,CADLeadWWID,PIFLeadWWID,Comments,WWID,StartDate,EndDate,core,CreatedOn,UpdatedOn,RefBoardID,RefBoardName) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
	val = (designtype, boardstate, boardtrack, timeline,platform,sku,memorytype, deslead, DesignManager, cadlead,piflead,com,user_wwid,start,end,core,t,t,boardid_ref,ref_design_name)
	execute_query(sql,val)
	
	sql = "SELECT LAST_INSERT_ID()" #Returns request ID for BoardDetailsRequest.
	deid = str(execute_query_sql(sql)[0][0])

	# log table
	try:
		log_notes = 'User has created new design request. Request ID: '+str(deid)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Design Request',0,deid,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")

	if combination_design:
		
		design_request_IDs += ',' + deid

		if combination_design == "yes":
			sql = "INSERT INTO DesignCombination(DesignTypeID,PlatformID,SKUID,MemTypeID,core,BoardRequestIDs,Verified,VerifiedWWID) VALUES (%s,%s,%s,%s,%s,%s,%s,%s)"
			val = (designtype,platform,sku,memorytype,core,design_request_IDs,True,user_wwid)
			execute_query(sql,val)

	email_list.append(user_email[0][0])


	for email in admin_email:
		email_list.append(email[0])
		
	email_list = sorted(set(email_list), reverse=True)

	if boardtrack2 == "Yes":
		fast_track_content = '<span style="font-weight:bold;color:red;">Yes</span>'
	else:
		fast_track_content = 'No'


	message = '''
		   New Design Request has been raised by '''+ name+'''.<br><br>

		<span style="color: #0718f7">Design Type: </span>''' + designtype2

	if boardid_ref in [0,"0"]:
		message += '''<br><span style="color: #0718f7">Reference Design: </span>''' + str(ref_design_name) + ''' <br>'''
	else:
		message += '''<br><span style="color: #0718f7">Reference Design: </span>[ID: ''' + str(boardid_ref) + '''] - '''+str(ref_design_name)+''' <br>'''

	message += '''<br><span style="color: #0718f7">FastTrack Review Needed: </span>''' + fast_track_content + ''' <br> 
		<span style="color: #0718f7">Review Phase: </span>''' + timeline2 + ''' <br> 
		<span style="color: #0718f7">Platform: </span>''' + platform2 + ''' <br>
		<span style="color: #0718f7">SKU: </span>''' + sku2 + ''' <br>
		<span style="color: #0718f7">Core: </span>''' +core2 +''' <br>
		<span style="color: #0718f7">Memory Type: </span>''' + memorytype2 + ''' <br>
		<span style="color: #0718f7">Design Lead: </span>''' + DesignLeadName + ''' <br>
		<span style="color: #0718f7">Design Manager: </span>''' + DesignManager  + ''' <br>
		<span style="color: #0718f7">LayoutLead/Manager: </span>''' + CADlead + ''' <br>
		<span style="color: #0718f7">PIF Lead: </span>''' + PIFlead + ''' <br>
		<span style="color: #0718f7">Review Start Date: </span>''' + get_work_week_str_fmt(start)  + ''' <br>
		<span style="color: #0718f7">Tape-Out Date: </span>''' + get_work_week_str_fmt(end) + ''' <br>
		<span style="color: #0718f7">Comments: </span>''' + comments + ''' <br><br>
		Regards,<br>
		ERAM.
		    '''  #####mail to the requester

	subject = "[Request ID:"+deid+"] New Design Review Request Raised"
	email_list = sorted(set(email_list), reverse=True)
	for e in email_list:
		send_mail(e,subject,message,email_list)

	return render("submitted.html",username=username,user_role_name=user_role_name,region_name=region_name) #change to a new html with submitted successfully

@app.route('/download_files_attachment_by_fileid',methods = ['POST', 'GET'])
def download_files_attachment_by_fileid():
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	fileid=request.form.get("fileid")
	filename=request.form.get("filename")
	sql="SELECT a.Files FROM UploadFileStorage a WHERE a.FileID=%s"
	val=(fileid,)
	d=execute_query(sql,val)
	if(d != ()):
		return send_file(BytesIO(d[0][0]),attachment_filename=filename,as_attachment=True)
	else:
		return render('error_custom.html',error='No Files are available to download.',username=username,user_role_name=user_role_name,region_name=region_name)

@app.route('/download_files_attachment_by_fileid_zip',methods = ['POST', 'GET'])
def download_files_attachment_by_fileid_zip():
	fileid=request.form.get("fileid")
	filename=request.form.get("filename")
	boardid=request.form.get("boardid")
	fileid_list = fileid.split(',')

	archive_name = 'test_file.zip'
	with zipfile.ZipFile(archive_name, 'w', zipfile.ZIP_DEFLATED) as file:
		pass

	# Board File
	if (fileid_list[0] is None) or (fileid_list[0] == ''):
		pass
	else:
		sql="SELECT a.Files FROM UploadFileStorage a WHERE a.FileID=%s"
		val=(fileid_list[0],)
		d=execute_query(sql,val)
		if(d != ()):
			sql="SELECT a.BoardFileName FROM UploadSignOffFiles a WHERE a.BoardID = %s AND a.BoardFileID=%s LIMIT 1"
			val=(boardid,fileid_list[0])
			file_name_rs=execute_query(sql,val)

			with zipfile.ZipFile(archive_name, 'a') as new_zip:
				fname = 'Board_File_'+file_name_rs[0][0]
				new_zip.writestr(fname,BytesIO(d[0][0]).read())


	# Schematics File
	if (fileid_list[1] is None) or (fileid_list[1] == ''):
		pass
	else:
		sql="SELECT a.Files FROM UploadFileStorage a WHERE a.FileID=%s"
		val=(fileid_list[1],)
		d=execute_query(sql,val)
		if(d != ()):
			sql="SELECT a.SchematicsName FROM UploadSignOffFiles a WHERE a.BoardID = %s AND a.SchematicsFileID=%s LIMIT 1"
			val=(boardid,fileid_list[1])
			file_name_rs=execute_query(sql,val)

			with zipfile.ZipFile(archive_name, 'a') as new_zip:
				fname = 'Schematics_File_'+file_name_rs[0][0]
				new_zip.writestr(fname,BytesIO(d[0][0]).read())


	# Stackup_File
	if (fileid_list[2] is None) or (fileid_list[2] == ''):
		pass
	else:
		sql="SELECT a.Files FROM UploadFileStorage a WHERE a.FileID=%s"
		val=(fileid_list[2],)
		d=execute_query(sql,val)
		if(d != ()):
			sql="SELECT a.StackupFileName FROM UploadSignOffFiles a WHERE a.BoardID = %s AND a.StackupFileID=%s LIMIT 1"
			val=(boardid,fileid_list[2])
			file_name_rs=execute_query(sql,val)

			with zipfile.ZipFile(archive_name, 'a') as new_zip:
				fname = 'Stackup_File_'+file_name_rs[0][0]
				new_zip.writestr(fname,BytesIO(d[0][0]).read())


	# Lenght_Report_File
	if (fileid_list[3] is None) or (fileid_list[3] == ''):
		pass
	else:
		sql="SELECT a.Files FROM UploadFileStorage a WHERE a.FileID=%s"
		val=(fileid_list[3],)
		d=execute_query(sql,val)
		if(d != ()):
			sql="SELECT a.LengthReportFileName FROM UploadSignOffFiles a WHERE a.BoardID = %s AND a.LengthReportFileID=%s LIMIT 1"
			val=(boardid,fileid_list[3])
			file_name_rs=execute_query(sql,val)

			with zipfile.ZipFile(archive_name, 'a') as new_zip:
				fname = 'Lenght_Report_File_'+file_name_rs[0][0]
				new_zip.writestr(fname,BytesIO(d[0][0]).read())


	# Others_File
	if (fileid_list[4] is None) or (fileid_list[4] == ''):
		pass
	else:
		sql="SELECT a.Files FROM UploadFileStorage a WHERE a.FileID=%s"
		val=(fileid_list[4],)
		d=execute_query(sql,val)
		if(d != ()):
			sql="SELECT a.OthersFileName FROM UploadSignOffFiles a WHERE a.BoardID = %s AND a.OthersFileID=%s LIMIT 1"
			val=(boardid,fileid_list[4])
			file_name_rs=execute_query(sql,val)

			with zipfile.ZipFile(archive_name, 'a') as new_zip:
				fname = 'Others_File_'+file_name_rs[0][0]
				new_zip.writestr(fname,BytesIO(d[0][0]).read())

	return send_file(archive_name,attachment_filename="Board_"+boardid+"_all_files.zip",as_attachment=True)

@app.route('/download_design_files',methods = ['POST', 'GET'])
def download_design_files():
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	board=request.form.get("download_board")
	sql="SELECT b.Files,a.FileName FROM UploadSignOffFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.FileID WHERE a.BoardID=%s ORDER BY a.Insert_Time DESC LIMIT 1"
	val=(board,)
	d=execute_query(sql,val)
	if(d != ()):
		return send_file(BytesIO(d[0][0]),attachment_filename="Board"+board+"_"+d[0][1],as_attachment=True)
	else:
		return render('error_custom.html',error='No Files are available to download.',username=username,user_role_name=user_role_name,region_name=region_name)


@app.route('/download_design_files_new',methods = ['POST', 'GET'])
def download_design_files_new():
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	board=request.form.get("download_board")
	file_number=request.form.get("file_number")

	if file_number:
		file_number = str(file_number)
	else:
		file_number = ''

	sql="SELECT b.Files,a.FileName FROM UploadDesignFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.FileID WHERE a.BoardID=%s"
	val=(board,)
	d=execute_query(sql,val)
	if(d != ()):

		if d[0][1] is None:

			sql="SELECT a.BoardFileName,b.Files,a.SchematicsName,c.Files,a.StackupFileName,d.Files,a.LengthReportFileName,e.Files,a.OthersFileName,f.Files FROM UploadSignoffLatestFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.BoardFileID LEFT JOIN UploadFileStorage c ON c.FileID=a.SchematicsFileID LEFT JOIN UploadFileStorage d ON d.FileID=a.StackupFileID LEFT JOIN UploadFileStorage e ON e.FileID=a.LengthReportFileID LEFT JOIN UploadFileStorage f ON f.FileID=a.OthersFileID  WHERE a.BoardID=%s"
			val=(board,)
			rs_files=execute_query(sql,val)

			if rs_files != ():
				if (rs_files[0][1]) and (file_number == '1'):
					return send_file(BytesIO(rs_files[0][1]),attachment_filename="Board_"+board+"_Board_File_"+rs_files[0][0],as_attachment=True)

				if (rs_files[0][3]) and (file_number == '2'):
					return send_file(BytesIO(rs_files[0][3]),attachment_filename="Board_"+board+"_Schematic_File_"+rs_files[0][2],as_attachment=True)

				if (rs_files[0][5]) and (file_number == '3'):
					return send_file(BytesIO(rs_files[0][5]),attachment_filename="Board_"+board+"_Stackup_File_"+rs_files[0][4],as_attachment=True)

				if (rs_files[0][7]) and (file_number == '4'):
					return send_file(BytesIO(rs_files[0][7]),attachment_filename="Board_"+board+"_LengthReport_File_"+rs_files[0][6],as_attachment=True)

				if (rs_files[0][9]) and (file_number == '5'):
					return send_file(BytesIO(rs_files[0][9]),attachment_filename="Board_"+board+"_Others_File_"+rs_files[0][8],as_attachment=True)


			return render('error_custom.html',error='Files are not available to download.',username=username,user_role_name=user_role_name,region_name=region_name)

		else:
			return send_file(BytesIO(d[0][0]),attachment_filename="Board"+board+".zip",as_attachment=True)
	else:
		return render('error_custom.html',error='No Files are available to download.',username=username,user_role_name=user_role_name,region_name=region_name)

@app.route('/download_design_files_all',methods = ['POST', 'GET'])
def download_design_files_all():
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	wwid=session.get('wwid')
	board=request.form.get("download_board")

	archive_name = 'test_file.zip'
	with zipfile.ZipFile(archive_name, 'w', zipfile.ZIP_DEFLATED) as file:
		pass

	sql="SELECT a.BoardFileName,b.Files,a.SchematicsName,c.Files,a.StackupFileName,d.Files,a.LengthReportFileName,e.Files,a.OthersFileName,f.Files FROM UploadSignoffLatestFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.BoardFileID LEFT JOIN UploadFileStorage c ON c.FileID=a.SchematicsFileID LEFT JOIN UploadFileStorage d ON d.FileID=a.StackupFileID LEFT JOIN UploadFileStorage e ON e.FileID=a.LengthReportFileID LEFT JOIN UploadFileStorage f ON f.FileID=a.OthersFileID  WHERE a.BoardID=%s"
	val=(board,)
	rs_files=execute_query(sql,val)

	if rs_files != ():

		if rs_files[0][1]:
			with zipfile.ZipFile(archive_name, 'a') as new_zip:
				fname = 'Board_File_'+rs_files[0][0]
				new_zip.writestr(fname,BytesIO(rs_files[0][1]).read())

		if rs_files[0][3]:
			with zipfile.ZipFile(archive_name, 'a') as new_zip:
				fname = 'Schematics_File_'+rs_files[0][2]
				new_zip.writestr(fname,BytesIO(rs_files[0][3]).read())

		if rs_files[0][5]:
			with zipfile.ZipFile(archive_name, 'a') as new_zip:
				fname = 'Stackup_File_'+rs_files[0][4]
				new_zip.writestr(fname,BytesIO(rs_files[0][5]).read())

		if rs_files[0][7]:
			with zipfile.ZipFile(archive_name, 'a') as new_zip:
				fname = 'Lenght_Report_File_'+rs_files[0][6]
				new_zip.writestr(fname,BytesIO(rs_files[0][7]).read())

		if rs_files[0][9]:
			with zipfile.ZipFile(archive_name, 'a') as new_zip:
				fname = 'Others_File_'+rs_files[0][8]
				new_zip.writestr(fname,BytesIO(rs_files[0][9]).read())

		return send_file(archive_name,attachment_filename="Board_"+board+"_all_files.zip",as_attachment=True)

	else:
		return render('error_custom.html',error='No Files are available to download.',username=username,user_role_name=user_role_name,region_name=region_name)

@app.route('/review_design.html',methods = ['POST', 'GET'])    ##method for admin to review board request
def review_design():

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'review_design'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	wwid=session.get('wwid')
	name = session.get('username')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	query = "SELECT a.RequestID,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,c.Username,DesignManagerWWID,d.Username,e.Username,Comments,StartDate,EndDate,core,a.WWID,b.Username,IF(a.ReviewTimelineID=1,'Rev0p6','Rev1p0'),a.BoardTrackID,a.ReviewTimelineID,a.RefBoardID,a.RefBoardName FROM BoardDetailsRequest a NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) LEFT JOIN HomeTable b ON a.WWID = b.WWID LEFT JOIN HomeTable c ON a.DesignLeadWWID = c.WWID LEFT JOIN HomeTable d ON a.CADLeadWWID = d.WWID LEFT JOIN HomeTable e ON a.PIFLeadWWID = e.WWID WHERE BoardStateID in (3,4,5) ORDER BY a.RequestID ASC"
	details_rs = execute_query_sql(query)

	details = []
	for row in details_rs:
		temp = []
		temp.append(row[0])
		temp.append(row[1])
		temp.append(row[2])
		temp.append(row[3])
		temp.append(row[4])
		temp.append(row[5])
		temp.append(row[6])
		temp.append(row[7])
		temp.append(row[8])
		temp.append(row[9])
		temp.append(row[10])
		temp.append(row[11])
		temp.append(row[12])
		temp.append(row[13])
		temp.append(row[14])
		temp.append(row[15])
		temp.append(row[16])
		temp.append(row[17])
		temp.append(row[18])

		if row[19] == 1: # fast track request
			#temp.append(get_work_week_addition(date_value=row[13],no_of_days=2))
			temp.append(get_work_week_addition(date_value=row[13],no_of_days=1))
		else:
			if row[20] == 1: #Rev0p6
				#temp.append(get_work_week_addition(date_value=row[13],no_of_days=3))
				temp.append(get_work_week_addition(date_value=row[13],no_of_days=2))
			else:
				#temp.append(get_work_week_addition(date_value=row[13],no_of_days=4))
				temp.append(get_work_week_addition(date_value=row[13],no_of_days=3))

		# ww - start date
		temp.append(get_work_week_fun_with_year(date_value=row[13]))	#20
		temp.append(get_work_week_fun_with_year(date_value=row[14]))	#21

		if row[22] != "":												#22 - Reference design
			if row[21] in [0,"0"]:
				temp.append(row[22])
			else:
				temp_text = "ID: "+str(row[21])+" - "+str(row[22])
				temp.append(temp_text)
		else:
			temp.append("-")

		print(temp)
		details.append(temp)


	sql="SELECT UserName FROM HomeTable WHERE IsActive = 1 ORDER BY UserName"
	usernames=execute_query_sql(sql)

	sql="select RoleID from HomeTable where WWID=%s"
	val = (wwid,)
	role_id=execute_query(sql,val)[0][0]

	if role_id==14:
		eram_mgt_access="yes"
	else:
		eram_mgt_access="no"

	is_elec_owner = False
	if(role_id == 4 or role_id == 1 or role_id == 2 or role_id == 10 or role_id == 12 or role_id == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role_id == 3 or role_id == 5 or role_id == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role_id == 7 or role_id == 8 or role_id == 9):
		is_layout_owner = True

	sql="Select AdminAccess from HomeTable where WWID=%s"
	val=(wwid,)
	hasadminaccess=execute_query(sql,val)

	if(hasadminaccess[0][0]=="yes"):
		is_admin=True
	else:
		is_admin=False

	sql="select RoleID from HomeTable where WWID=%s "
	val = (wwid,)
	role_id=execute_query(sql,val)
	if (role_id[0][0] == 14):
		mgt_access=True
	else:
		mgt_access=False

	return render("review_design.html",is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,mgt_access=mgt_access,details=details,name=name,usernames=usernames,is_admin=is_admin,eram_mgt_access=eram_mgt_access,username=username,user_role_name=user_role_name,region_name=region_name)


@app.route('/review_design_eram_mgt.html',methods = ['POST', 'GET'])    ##method for admin to review board request
def review_design_eram_mgt():
	wwid = session.get('wwid')
	name = session.get('username')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	query = "SELECT RequestID,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,DesignLeadWWID,DesignManagerWWID,CADLeadWWID,PIFLeadWWID,Comments,StartDate,EndDate,core FROM BoardDetailsRequest NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) WHERE BoardStateID =3 ORDER BY RequestID ASC"
	details = execute_query_sql(query)

	query = "SELECT b.RequestID,a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.WWID ORDER BY b.RequestID ASC"
	submitby = execute_query_sql(query)
	submitby_list = []

	for i in range(0,len(details)):
		for j in range(0,len(submitby)):
			if details[i][0] == submitby[j][0]:
				submitby_list.append(submitby[j][1])

	query = "SELECT b.RequestID,a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.DesignLeadWWID ORDER BY b.RequestID ASC"
	designlist = execute_query_sql(query)
	designlead_list = []

	for i in range(0,len(details)):
		for j in range(0,len(designlist)):
			if details[i][0] == designlist[j][0]:
				designlead_list.append(designlist[j][1])

	query = "SELECT b.RequestID,a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.CADLeadWWID  ORDER BY b.RequestID ASC"
	cadlist = execute_query_sql(query)
	cadlead_list = []

	for i in range(0,len(details)):
		for j in range(0,len(cadlist)):
			if details[i][0] == cadlist[j][0]:
				cadlead_list.append(cadlist[j][1])

	query = "SELECT b.RequestID,a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.PIFLeadWWID  ORDER BY b.RequestID ASC"
	piflist = execute_query_sql(query)
	piflead_list = []

	for i in range(0,len(details)):
		for j in range(0,len(piflist)):
			if details[i][0] == piflist[j][0]:
				piflead_list.append(piflist[j][1])

	sql="select UserName from HomeTable"
	usernames=execute_query_sql(sql)

	sql="select RoleID from HomeTable where WWID=%s"
	val = (wwid,)
	role_id=execute_query(sql,val)
	if role_id==14:
		eram_mgt_access="yes"
	else:
		eram_mgt_access="no"

	sql="Select AdminAccess from HomeTable where WWID=%s"
	val=(wwid,)
	hasadminaccess=execute_query(sql,val)

	if(hasadminaccess[0][0]=="yes"):
		admin=True
	else:
		admin=False

	return render("review_design_eram_mgt.html",details=details,name=name,designlead=designlead_list,cadlead=cadlead_list,piflead=piflead_list,usernames=usernames,admin=admin,eram_mgt_access=eram_mgt_access,submitby_list=submitby_list,username=username,user_role_name=user_role_name,region_name=region_name)

@app.route('/approve_design',methods = ['POST', 'GET'])
def approve_access():

	user_wwid = session.get('wwid')
	name = session.get('username')

	board_approve_id = request.form.get('board_approve_id')
	enddate = request.form.get('enddate')
	#boardstate="Design Team Projection"
	boardname=request.form.get('boardname')
	design_calendar_status=request.form.get('design_calendar_status')

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	# log table
	try:
		log_notes = 'User has approved new design request. Request ID: '+str(board_approve_id)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Design Request',0,board_approve_id,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	sql = "Update BoardDetailsRequest SET BoardStateID = 1 WHERE RequestID = %s"#Approved
	val = (board_approve_id,)
	execute_query(sql,val)

	query = "SELECT * FROM BoardDetailsRequest WHERE BoardStateID='1' AND RequestID=%s"
	val=(board_approve_id,)
	approved = execute_query(query,val)
	approved_list = []
	for i in approved:
		approved_list.append(i)

	sql = "INSERT INTO BoardDetails(BoardName,DesignTypeID,BoardStateID,BoardTrackID,ReviewTimelineID,PlatformID,SKUID,MemTypeID,DesignLeadWWID,DesignManagerWWID,CADLeadWWID,PIFLeadWWID,core,DocStoredLocation,ClosureComment,CreatedOn,UpdatedOn,RefBoardID,RefBoardName) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,'',%s,%s,%s,%s)"
	val=(boardname,approved_list[0][1],approved_list[0][2],approved_list[0][3],approved_list[0][4],approved_list[0][5],approved_list[0][6],approved_list[0][7],approved_list[0][8],approved_list[0][9],approved_list[0][10],approved_list[0][11],approved_list[0][16],"location",t,t,approved_list[0][19],approved_list[0][20])
	execute_query(sql,val)

	sql = 'Update DesignType SET DesignStatus = "Approved" WHERE DesignTypeID = %s'
	val = (approved_list[0][1],)																###UPDATING DESIGN TYPE ENTRY TO APPROVED
	execute_query(sql, val)

	sql = 'Update MemType SET MemTypeStatus = "Approved" WHERE MemTypeID = %s'
	val = (approved_list[0][7],)  ###UPDATING DESIGN TYPE ENTRY TO APPROVED
	execute_query(sql, val)

	sql = 'Update SUK SET SKUStatus = "Approved" WHERE SKUID = %s'
	val = (approved_list[0][6],)  ###UPDATING DESIGN TYPE ENTRY TO APPROVED
	execute_query(sql, val)

	sql = 'Update Platform SET PlatformStatus = "Approved" WHERE PlatformID = %s'
	val = (approved_list[0][5],)  ###UPDATING DESIGN TYPE ENTRY TO APPROVED
	execute_query(sql, val)


	sql = "SELECT WWID FROM BoardDetailsRequest WHERE RequestID=%s"
	val=(board_approve_id,)
	approved_wwid = execute_query(sql,val)


	sql = "SELECT EmailID FROM RequestAccess WHERE WWID=%s"
	val = (approved_wwid,) 											 ###### CHANGE THIS TO session.get('wwid')
	user_email = execute_query(sql,val)

	sql="select BoardID from BoardDetails where BoardName=%s"
	val=(boardname,)
	bid=execute_query(sql,val)

	sql = "INSERT INTO RequestMap VALUES (%s,%s)"
	val = (board_approve_id,bid[0][0])
	execute_query(sql,val)

	query = "SELECT RequestID,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,DesignLeadWWID,DesignManagerWWID,CADLeadWWID,PIFLeadWWID,Comments,StartDate,EndDate,core,RefBoardID,RefBoardName FROM BoardDetailsRequest NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) WHERE RequestID=%s"
	val = (board_approve_id,)
	details = execute_query(query, val)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.DesignLeadWWID and b.RequestID=%s"
	val = (board_approve_id,)
	deslead = execute_query(query, val)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.CADLeadWWID and b.RequestID=%s"
	val = (board_approve_id,)
	cad = execute_query(query, val)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.PIFLeadWWID and b.RequestID=%s"
	val = (board_approve_id,)
	pif = execute_query(query, val)


	leads=[]
	sql="select DesignLeadWWID,CADLeadWWID,PIFLeadWWID from BoardDetailsRequest where RequestID=%s"
	val=(board_approve_id,)
	lead=execute_query(sql,val)
	for i in lead:
		leads.append(i[0])
	sql="select EmailID from HomeTable where WWID in %s"
	val=(leads,)
	emailslead=execute_query(sql,val)

	sql = "SELECT EmailID FROM HomeTable WHERE AdminAccess= %s"
	val = ('yes',)
	admin_email = execute_query(sql,val)

	if details[0][3] == "Yes":
		fast_track_content = '<span style="font-weight:bold;color:red;">Yes</span>'
	else:
		fast_track_content = 'No'

	message = '''
		<h4>Design Review Request Approved by ERAM.</h4>
		<h5>Design Review Projection Details: </h5>
		<span style="color: #0718f7">Design ID: </span>''' + str(bid[0][0]) + '''<br>
		<span style="color: #0718f7">Design Name: </span>''' + boardname + '''<br><br>'''
		
	if details[0][17] != "":
		if details[0][16] not in [0,"0"]:
			message += '''<span style="color: #0718f7">Reference Design ID: </span>''' + str(details[0][16]) + '''<br>'''
			message += '''<span style="color: #0718f7">Reference Design Name: </span>''' + str(details[0][17]) + '''<br><br>'''
		else:
			message += '''<span style="color: #0718f7">Reference Design Name: </span>''' + str(details[0][17]) + '''<br><br>'''

	message += '''<span style="color: #0718f7">Board State: </span>''' + details[0][2] + ''' <br>
		<span style="color: #0718f7">Design Type: </span>''' + details[0][1] + ''' <br>
		<span style="color: #0718f7">Platform: </span>''' + details[0][5] + ''' <br>
		<span style="color: #0718f7">SKU: </span>''' + details[0][6] + ''' <br>
		<span style="color: #0718f7">Core: </span>''' + details[0][15] + ''' <br>
		<span style="color: #0718f7">Memory Type: </span>''' + details[0][7] + ''' <br><br>

		<span style="color: #0718f7">FastTrack Review Requested: </span>''' + fast_track_content + ''' <br> 
		<span style="color: #0718f7">Review Phase: </span>''' + details[0][4] + ''' <br> 
		<span style="color: #0718f7">Start Date: </span>''' + get_work_week_date_fmt(details[0][13]) + ''' <br>
		<span style="color: #0718f7">Tape Out Date: </span>''' + get_work_week_date_fmt(details[0][14]) + ''' <br><br>

		<span style="color: #0718f7">Design Lead: </span>''' + deslead[0][0] + ''' <br>
		<span style="color: #0718f7">Design Manager: </span>''' + str(details[0][9]) + ''' <br>
		<span style="color: #0718f7">Layout Lead/Manager: </span>''' + cad[0][0] + ''' <br>
		<span style="color: #0718f7">PIF Lead: </span>''' + pif[0][0] + ''' <br>
		<span style="color: #0718f7">Comments: </span>''' + details[0][12] + ''' <br><br>
		Regards,<br>
		ERAM.

		    '''  #####mail to the requester

	subject = "[Request ID: "+board_approve_id+" ] Design Review Request Approved"

	email_list = []
	email_list.append(user_email[0][0])
	for j in emailslead:
		email_list.append(j[0])

	for j in admin_email:
		email_list.append(j[0])

	email_list = sorted(set(email_list), reverse=True)
	for i in email_list:
		send_mail(i, subject, message,email_list)

	sql="select StartDate from BoardDetailsRequest where RequestID=%s"
	val=(board_approve_id,)
	startdate=execute_query(sql,val)

	sql = "INSERT INTO DesignCalendar VALUES(%s,%s,%s,0000-00-00,0000-00-00,%s,null) "
	val = (bid[0][0],startdate[0][0],enddate,design_calendar_status,)
	execute_query(sql, val)


	sql="insert into ScheduleTable values(%s,%s)"
	val=(bid[0][0],3)
	execute_query(sql,val)

	return redirect(url_for('review_design', _external=True))

@app.route('/check_boards',methods = ['POST', 'GET'])
def check_boards():
	clist={}
	boardname=request.form.get("boardname")
	sql="select * FROM BoardDetails WHERE BoardName=%s"
	val=(boardname,)
	nameexists=execute_query(sql,val)
	if(nameexists==()):
		clist["exists"]='NO'
	else:
		clist["exists"]='YES'
	return jsonify(clist)

@app.route('/reject_design',methods = ['POST', 'GET'])
def reject_access():
	rejectreason=request.form.get("reject_reason")
	user_wwid=session.get('wwid')
	username=session.get('username')
	board_reject_id = request.form.get('rejectid')

	# log table
	try:
		log_notes = 'User has Rejected new design request. <br>Request ID: '+str(board_reject_id)+'<br>Comments: '+str(rejectreason)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Design Request',0,board_reject_id,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	sql = "SELECT Comments FROM BoardDetailsRequest WHERE RequestID = %s"
	val = (board_reject_id,)
	com = execute_query(sql,val)[0][0]

	com2 = username + ": " + rejectreason + ",Last Comment:" + com

	sql = "Update BoardDetailsRequest SET BoardStateID = 2, Comments = %s WHERE RequestID =%s"
	val = (com2,board_reject_id)
	execute_query(sql,val)

	sql = "SELECT WWID FROM BoardDetailsRequest WHERE RequestID=%s"
	val = (board_reject_id,)
	reject_wwid = execute_query(sql, val)


	sql = "SELECT EmailID FROM RequestAccess WHERE WWID=%s"
	val = (reject_wwid,)                           ###### CHANGE THIS TO session.get('wwid')
	user_email = execute_query(sql,val)

	leads = []
	sql = "select DesignLeadWWID,CADLeadWWID,PIFLeadWWID from BoardDetailsRequest where RequestID=%s"
	val = (board_reject_id,)
	lead = execute_query(sql, val)
	for i in lead:
		leads.append(i[0])
	sql = "select EmailID from HomeTable where WWID in %s"
	val = (leads,)
	emailslead = execute_query(sql, val)
	message = '''
			  Design Request has been Rejected!<br><br>

		Reason for Rejection: ''' + rejectreason + '''<br>
			   '''  #####mail to the requester
	subject = "[Request ID :"+ board_reject_id+"] Design Review Request Rejected"

	email_list = []
	send_mail(user_email[0][0], subject, message,email_list)
	for j in emailslead:
		send_mail(j[0], subject, message)

	query = "SELECT RequestID,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,DesignLeadWWID,DesignManagerWWID,CADLeadWWID,PIFLeadWWID,Comments FROM BoardDetailsRequest NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) WHERE BoardStateID in (3,4,5) ORDER BY RequestID ASC"
	details = execute_query_sql(query)
	details_list = []
	for i in range(len(details)):
		role = details[len(details) - i - 1]
		details_list.append(role)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.DesignLeadWWID ORDER BY b.RequestID ASC"
	designlist = execute_query_sql(query)
	designlead_list = []
	for i in range(len(designlist)):
		role = designlist[len(designlist) - i - 1]
		designlead_list.append(role)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.CADLeadWWID ORDER BY b.RequestID ASC"
	cadlist = execute_query_sql(query)
	cadlead_list = []
	for i in range(len(cadlist)):
		role = cadlist[len(cadlist) - i - 1]
		cadlead_list.append(role)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.PIFLeadWWID ORDER BY b.RequestID ASC"
	piflist = execute_query_sql(query)
	piflead_list = []
	for i in range(len(piflist)):
		role = piflist[len(piflist) - i - 1]
		piflead_list.append(role)

	return redirect(url_for('review_design', _external=True))

@app.route('/fwd_mgmt',methods = ['POST', 'GET'])
def fwd_mgmt():
	fwd_id=request.form.get("fwd_id")
	usernames = request.form.getlist("usernames")
	fwd_comment=request.form.get("forward_comment")
	user=session.get('username')

	sql = "SELECT Comments FROM BoardDetailsRequest WHERE RequestID = %s"
	val = (fwd_id,)
	com = execute_query(sql,val)[0][0]

	com2 = user + ": " + fwd_comment + ",Last Comment:" + com

	sql = "Update BoardDetailsRequest SET Comments = %s WHERE RequestID =%s"
	val = (com2,fwd_id)
	execute_query(sql,val)

	eids=[]
	if(usernames):
		sql = "select EmailID from HomeTable where UserName in %s"
		val = (usernames,)
		eids = execute_query(sql, val)

	query = "SELECT RequestID,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,DesignLeadWWID,DesignManagerWWID,CADLeadWWID,PIFLeadWWID,Comments,StartDate,EndDate FROM BoardDetailsRequest NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) WHERE RequestID=%s"
	val=(fwd_id,)

	details = execute_query(query,val)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.DesignLeadWWID and b.RequestID=%s"
	val = (fwd_id,)
	deslead =  execute_query(query,val)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.CADLeadWWID and b.RequestID=%s"
	val = (fwd_id,)
	cad =  execute_query(query,val)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.PIFLeadWWID and b.RequestID=%s"
	val = (fwd_id,)
	pif =  execute_query(query,val)

	sql = "Update BoardDetailsRequest SET BoardStateID = 3 WHERE RequestID = %s"
	val = (fwd_id,)
	execute_query(sql, val)

	name = session.get('username')

	if details[0][3] == "Yes":
		fast_track_content = '<span style="font-weight:bold;color:red;">Yes</span>'
	else:
		fast_track_content = 'No'

	subject = "[Request ID: "+ fwd_id+"] Design Review - Management Approval Requested"
	message = ''' Management Approval Requested By ''' + name +''' <br><br>


		Comments by admin: ''' +fwd_comment+ '''<br><br>
		Design Review Projection Details <br>

		Design Type: ''' + details[0][1] + ''' <br>
		Board State: ''' + details[0][2] + ''' <br>
		FastTrack Review Requested: ''' + fast_track_content + ''' <br> 
		Review Phase: ''' + details[0][4] + ''' <br> 
		Platform: ''' + details[0][5] + ''' <br>
		SKU: ''' + details[0][6] + ''' <br>
		Memory Type: ''' + details[0][7] + ''' <br>
		Design Lead: ''' + deslead[0][0] + ''' <br>
		Design Manager: ''' + str(details[0][9]) + ''' <br>
		Layout Lead/Manager: ''' + cad[0][0] + ''' <br>
		PIF Lead: ''' + pif[0][0] + ''' <br>
		Start Date: ''' + get_work_week_date_fmt(details[0][13]) + ''' <br>
		Tape Out Date: ''' + get_work_week_date_fmt(details[0][14]) + ''' <br>

		Comments: ''' + details[0][12] + ''' <br>	
'''
	sql="select EmailID from HomeTable where RoleID=14"
	emailmgmt=execute_query_sql(sql)
	email_list = []
	for i in range(len(eids)):
		if(eids[i][0]):
			send_mail(eids[i][0], subject, message,email_list)
	for j in emailmgmt:
		send_mail(j[0], subject, message,email_list)

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val = ('yes',)
	admin_email = execute_query(sql,val)

	send_mail(admin_email[0][0],subject,message,email_list)
	
	return redirect(url_for('review_design', _external=True))

@app.route('/accept_timeline',methods = ['POST', 'GET'])
def accept_timeline():
	reqid = request.form.get("removeid")
	reason = request.form.get("delete_reason")
	username = session.get('username')
	sql = "SELECT Comments FROM BoardDetailsRequest WHERE RequestID = %s"
	val = (reqid,)
	com = execute_query(sql,val)[0][0]

	com2 = username + ": " + reason + ",Last Comment:" + com

	# log table
	try:
		log_notes = 'User has Accepted Timlines for Request ID: '+str(reqid)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Design Request',0,reqid,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	sql = "Update BoardDetailsRequest SET BoardStateID = 5, Comments = %s WHERE RequestID =%s"
	val = (com2,reqid)
	execute_query(sql,val)

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val = ('yes',)
	admin_email = execute_query(sql,val)

	subject = "[Request ID :"+ reqid+"] Design Review - Timeline Accepted"
	message = ''' New Timeline proposed has been accepted by ''' + username +''' <br><br>
					<h2> Comments by User: ''' +com+ '''<br><br>'''	

	for i in admin_email:
		send_mail(i[0],subject,message)
	return redirect(url_for('request_review', _external=True))	

@app.route('/reject_timeline',methods = ['POST', 'GET'])
def reject_timeline():
	reqid = request.form.get("removeid")
	reason = request.form.get("delete_reason")
	username = session.get('UserName')
	st = request.form.get("start")
	ed = request.form.get("end")

	sql = "SELECT Comments FROM BoardDetailsRequest WHERE RequestID = %s"
	val = (reqid,)
	com = execute_query(sql,val)[0][0]

	com2 = username + ": " + reason + ",Last Comment:" + com

	# log table
	try:
		log_notes = 'User has Rejected Timlines for Request ID: '+str(reqid)+'<br>Comments: '+str(reason)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Design Request',0,reqid,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	sql = "Update BoardDetailsRequest SET BoardStateID = 5, Comments = %s, StartDate = %s, EndDate=%s WHERE RequestID =%s"
	val = (com2,st,ed,reqid)
	execute_query(sql,val)

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val = ('yes',)
	admin_email = execute_query(sql,val)

	subject = "[Request ID :"+ reqid+"] Design Review - Timeline Rejected"
	message = ''' New Timeline proposed has been rejected by ''' + username +''' <br><br>
				  Comments By User:'''+reason+'''<br>
				  New Start Date: ''' + get_work_week_str_fmt(st) + '''<br>
				  New End Date :'''+ get_work_week_str_fmt(ed) +'''<br>'''	

	for i in admin_email:
		send_mail(i[0],subject,message)

	return redirect(url_for('request_review', _external=True))		

@app.route('/reconsider_timeline',methods = ['POST', 'GET'])
def reconsider_timeline():
	startdate=request.form.get("rec_start")
	rec_id=request.form.get("rec_id")
	usernames = request.form.getlist("usernames")
	rec_comment=request.form.get("rec_comment")
	eids=[]

	if (usernames):
		sql = "select EmailID from HomeTable where UserName in %s"
		val = (usernames,)
		eids = execute_query(sql, val)

	# log table
	try:
		log_notes = 'User has Reconsidered Timlines for Request ID: '+str(rec_id)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Design Request',0,rec_id,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	sql = "Update BoardDetailsRequest SET BoardStateID = 4 WHERE RequestID = %s"
	val = (rec_id,)
	execute_query(sql, val)

	query = "SELECT RequestID,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,DesignLeadWWID,DesignManagerWWID,CADLeadWWID,PIFLeadWWID,Comments,StartDate,EndDate FROM BoardDetailsRequest NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) WHERE RequestID=%s"
	val=(rec_id,)

	details = execute_query(query,val)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.DesignLeadWWID and b.RequestID=%s"
	val = (rec_id,)
	deslead =  execute_query(query,val)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.CADLeadWWID and b.RequestID=%s"
	val = (rec_id,)
	cad =  execute_query(query,val)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.PIFLeadWWID and b.RequestID=%s"
	val = (rec_id,)
	pif =  execute_query(query,val)

	sql = "Update BoardDetailsRequest SET StartDate = %s WHERE RequestID = %s"
	val = (startdate,rec_id)
	execute_query(sql, val)

	if details[0][3] == "Yes":
		fast_track_content = '<span style="font-weight:bold;color:red;">Yes</span>'
	else:
		fast_track_content = 'No'

	subject = "[Request ID: "+rec_id+"] Design Review Request - New Timeline Proposed"
	message = ''' <h5>New Timeline Proposed </h5>

			RequestID: ''' + rec_id + '''<br>
			New Start Date Proposed: ''' + get_work_week_str_fmt(startdate) + '''<br>
			Comments by admin: ''' + rec_comment + '''<br><br>

		Design Review Projection Details <br>	
			
		Design Type: ''' + details[0][1] + ''' <br>
		Board State: ''' + details[0][2] + ''' <br>
		FastTrack Review Requested: ''' + fast_track_content + ''' <br> 
		Review Phase: ''' + details[0][4] + ''' <br> 
		Platform: ''' + details[0][5] + ''' <br>
		SKU: ''' + details[0][6] + ''' <br>
		Memory Type: ''' + details[0][7] + ''' <br>
		Design Lead: ''' + deslead[0][0] + ''' <br>
		Design Manager: ''' + str(details[0][9]) + ''' <br>
		Layout Lead/Manager: ''' + cad[0][0] + ''' <br>
		PIF Lead: ''' + pif[0][0] + ''' <br>
		Start Date: ''' + get_work_week_date_fmt(details[0][13]) + ''' <br>
		Tape Out Date: ''' + get_work_week_date_fmt(details[0][14]) + ''' <br>

		Comments: ''' + details[0][12] + ''' <br><br>
		Regards,<br>
		ERAM.
	'''
	email_list = []
	for i in range(len(eids)):
		if(eids[i][0]):
			send_mail(eids[i][0], subject, message,email_list)

	user = session.get('username')
	sql = "SELECT Comments FROM BoardDetailsRequest WHERE RequestID = %s"
	val = (rec_id,)
	com = execute_query(sql,val)[0][0]

	com2 = user + ": " + rec_comment + ",Last Comment:" + com

	sql = "Update BoardDetailsRequest SET Comments = %s WHERE RequestID =%s"
	val = (com2,rec_id)
	execute_query(sql,val)

	sql = "SELECT WWID FROM BoardDetailsRequest WHERE RequestID=%s"
	val=(rec_id,)
	approved_wwid = execute_query(sql,val)[0][0]


	sql = "SELECT EmailID FROM RequestAccess WHERE WWID=%s"
	val = (approved_wwid,) 											 ###### CHANGE THIS TO session.get('wwid')
	user_email = execute_query(sql,val)

	email_list = []
	send_mail(user_email[0][0],subject,message,email_list)

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val = ('yes',)
	admin_email = execute_query(sql,val)

	send_mail(admin_email[0][0],subject,message,email_list)

	return redirect(url_for('review_design', _external=True))

@app.route('/des_req_comment',methods=['POST','GET'])
def des_req_comment():
	des_comment = request.form.get('des_comment')
	req_id=request.form.get('req_id')
	user = session.get('username')

	# log table
	try:
		log_notes = 'User has updated comments for Request ID: '+str(req_id)+'<br>Comments: '+str(des_comment)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Design Request',0,req_id,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	sql = "SELECT Comments FROM BoardDetailsRequest WHERE RequestID = %s"
	val = (req_id,)
	com = execute_query(sql,val)[0][0]

	com2 = user + ": " + des_comment + ",Last Comment:" + com

	sql = "UPDATE BoardDetailsRequest SET Comments = %s WHERE RequestID=%s"
	val=(com2,req_id)
	execute_query(sql,val)
	return redirect(url_for('request_review',_external=True))

@app.route('/rev_design_comment',methods=['POST','GET'])
def rev_design_comment():
	des_comment = request.form.get('des_comment')
	req_id=request.form.get('req_id')
	user = session.get('username')

	# log table
	try:
		log_notes = 'User has Updated comments for Request ID: '+str(req_id)+'<br>Comments: '+str(des_comment)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Design Request',0,req_id,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	sql = "SELECT Comments FROM BoardDetailsRequest WHERE RequestID = %s"
	val = (req_id,)
	com = execute_query(sql,val)[0][0]

	com2 = user + ": " + des_comment + ",Last Comment:" + com

	sql = "Update BoardDetailsRequest SET Comments = %s WHERE RequestID =%s"
	val = (com2,req_id)
	execute_query(sql,val)

	return redirect(url_for('review_design',_external=True))

@app.route('/eram_mgt_approve_design',methods=['POST','GET'])
def eram_mgt_approve_design():
	mgt_comment=request.form.get('eram_mgt_comid')
	req_id=request.form.get('board_approve_id')
	user=session.get('username')

	sql = "SELECT Comments FROM BoardDetailsRequest WHERE RequestID = %s"
	val = (req_id,)
	com = execute_query(sql,val)[0][0]

	com2 = user + ": " + mgt_comment + ",Last Comment:" + com

	sql = "Update BoardDetailsRequest SET BoardStateID = 6, Comments = %s WHERE RequestID =%s"
	val = (com2,req_id)
	execute_query(sql,val)

	sql = "SELECT WWID FROM BoardDetailsRequest WHERE RequestID=%s"
	val = (req_id,)
	approve_wwid = execute_query(sql, val)

	sql = "SELECT EmailID FROM RequestAccess WHERE WWID=%s"
	val = (approve_wwid,)                           ###### CHANGE THIS TO session.get('wwid')
	user_email = execute_query(sql,val)

	sql = "SELECT EmailID FROM HomeTable where AdminAccess='yes'"
	admin_list = execute_query_sql(sql)

	message = '''
			  Design Request has been Approved from ERAM Managemnet!<br><br>

		Reason for Approval: ''' + mgt_comment + '''<br>
			   '''  #####mail to the requester
	subject = "[Request ID :"+ req_id+"] Design Review Request Approved from Management"

	send_mail(user_email[0][0], subject, message)
	for j in admin_list:
		send_mail(j[0], subject, message)
		
	return redirect(url_for('review_design_eram_mgt',_external=True))

@app.route('/eram_mgt_reject_design', methods=['POST','GET'])
def eram_mgt_reject_design():
	mgt_comment=request.form.get('reject_reason')
	req_id=request.form.get('rejectid')
	user=session.get('username')

	sql = "SELECT Comments FROM BoardDetailsRequest WHERE RequestID = %s"
	val = (req_id,)
	com = execute_query(sql,val)[0][0]

	com2 = user + ": " + mgt_comment + ",Last Comment:" + com

	sql = "Update BoardDetailsRequest SET BoardStateID = 7, Comments = %s WHERE RequestID =%s"
	val = (com2,req_id)
	execute_query(sql,val)

	sql = "SELECT WWID FROM BoardDetailsRequest WHERE RequestID=%s"
	val = (req_id,)
	reject_wwid = execute_query(sql, val)

	sql = "SELECT EmailID FROM RequestAccess WHERE WWID=%s"
	val = (reject_wwid,)                           ###### CHANGE THIS TO session.get('wwid')
	user_email = execute_query(sql,val)

	sql = "SELECT EmailID FROM HomeTable where AdminAccess='yes'"
	admin_list = execute_query_sql(sql)

	message = '''
			  Design Request has been Rejected from ERAM Managemnet!<br><br>

		Reason for Rejection: ''' + mgt_comment + '''<br>
			   '''  #####mail to the requester
	subject = "[Request ID :"+ req_id+"] Design Review Request Rejected from Management"

	send_mail(user_email[0][0], subject, message)
	for j in admin_list:
		send_mail(j[0], subject, message)

	return redirect(url_for('review_design_eram_mgt',_external=True))

@app.route('/design_req_rev.html',methods = ['POST', 'GET'])			##method for reviewer to review board request
def request_review():

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'request_review'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	user_wwid = session.get('wwid')
	name = session.get('username')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	#query = "SELECT a.RequestID,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,c.Username,DesignManagerWWID,d.Username,e.Username,Comments,StartDate,EndDate,core,a.WWID,b.Username,IFNULL(f.BoardID,'') FROM BoardDetailsRequest a NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) LEFT JOIN HomeTable b ON a.WWID = b.WWID LEFT JOIN HomeTable c ON a.DesignLeadWWID = c.WWID LEFT JOIN HomeTable d ON a.CADLeadWWID = d.WWID LEFT JOIN HomeTable e ON a.PIFLeadWWID = e.WWID LEFT JOIN RequestMap f ON a.RequestID=f.RequestID WHERE a.WWID = %s OR a.DesignLeadWWID = %s OR a.CADLeadWWID = %s ORDER BY a.RequestID DESC"
	query = "SELECT a.RequestID,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,c.Username,DesignManagerWWID,d.Username,e.Username,Comments,StartDate,EndDate,core,a.WWID,b.Username,IFNULL(f.BoardID,''),IFNULL(st.ScheduleStatusID,0) FROM BoardDetailsRequest a NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) LEFT JOIN HomeTable b ON a.WWID = b.WWID LEFT JOIN HomeTable c ON a.DesignLeadWWID = c.WWID LEFT JOIN HomeTable d ON a.CADLeadWWID = d.WWID LEFT JOIN HomeTable e ON a.PIFLeadWWID = e.WWID LEFT JOIN RequestMap f ON a.RequestID=f.RequestID LEFT JOIN ScheduleTable st ON st.BoardID = f.BoardID WHERE a.WWID = %s OR a.DesignLeadWWID = %s OR a.CADLeadWWID = %s ORDER BY a.RequestID DESC"
	val=(user_wwid,user_wwid,user_wwid)
	details = execute_query(query,val)

	query = "SELECT Username FROM HomeTable WHERE IsActive = 1 AND RoleID='3'"
	des_manager = execute_query_sql(query)
	des_manager_list = []
	for i in range(len(des_manager)):
		user = des_manager[i]
		des_manager_list.append(user)

	query = "SELECT Username FROM HomeTable WHERE IsActive = 1 AND RoleID in ('7','8')"
	cadlead = execute_query_sql(query)
	cadleads = []
	for i in range(len(cadlead)):
		user = cadlead[i]
		cadleads.append(user)

	query = "SELECT Username FROM HomeTable WHERE IsActive = 1 AND RoleID='5'"
	deslead = execute_query_sql(query)
	desleads = []
	for i in range(len(deslead)):
		user = deslead[i]
		desleads.append(user)

	query = "SELECT BoardTrackName FROM BoardTrack"
	track = execute_query_sql(query)
	track_list = []
	for i in track:
		role = i[0]
		track_list.append(role)

	sql = "select UserName from HomeTable"
	usernames = execute_query_sql(sql)

	return render("design_req_rev.html",username=username,user_role_name=user_role_name,region_name=region_name,session=details,usernames=usernames,track=track_list,name=name,allcadleads=cadleads,alldesleads=desleads,des_manager = des_manager_list)


@app.route('/delete_req',methods = ['POST', 'GET'])			##method for requestor to delete board request
def delete_request():
	usernames = request.form.getlist("usernames")

	name = session.get('username')
	delcomment=request.form.get('delete_reason')
	user_wwid=session.get("wwid")
	removeid = request.form.get('removeid')

	sql="select a.UserName,a.WWID from HomeTable a,BoardDetailsRequest b where b.RequestID=%s and b.WWID=a.WWID "
	val=(removeid,)
	deleted_by=execute_query(sql,val)

	# log table
	try:
		log_notes = 'User has Deleted New Design Request. <br>Request ID: '+str(removeid)+'<br>Comments: '+str(delcomment)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Design Request',0,removeid,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	


	sql = "DELETE FROM BoardDetailsRequest WHERE RequestID= %s"
	val=(removeid,)
	execute_query(sql,val)
	eids=[]
	if (usernames):
		sql = "select EmailID from HomeTable where UserName in %s"
		val = (usernames,)
		eids = execute_query(sql, val)

	query = "SELECT RequestID,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,DesignLeadWWID,DesignManagerWWID,CADLeadWWID,PIFLeadWWID,Comments FROM BoardDetailsRequest NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) WHERE WWID=%s ORDER BY RequestID ASC"
	# change hardcoded wwid in WHERE to session.wwid  ###### CHANGE THIS TO session.get('wwid')
	val=(user_wwid,)
	current = execute_query(query,val)
	session_list = []
	for i in range(len(current)):
		user = current[len(current)-i-1]
		session_list.append(user)


	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.DesignLeadWWID ORDER BY b.RequestID ASC "
	designlist = execute_query_sql(query)
	designlead_list = []
	for i in range(len(designlist)):
		role = designlist[len(designlist) - i - 1]
		designlead_list.append(role)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.CADLeadWWID ORDER BY   b.RequestID ASC "
	cadlist = execute_query_sql(query)
	cadlead_list = []
	for i in range(len(cadlist)):
		role = cadlist[len(cadlist) - i - 1]
		cadlead_list.append(role)

	query = "SELECT a.Username FROM RequestAccess a, BoardDetailsRequest b WHERE a.WWID=b.PIFLeadWWID ORDER BY  b.RequestID ASC "
	piflist = execute_query_sql(query)
	piflead_list = []
	for i in range(len(piflist)):
		role = piflist[len(piflist) - i - 1]
		piflead_list.append(role)

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val = ('yes',)
	admin_email = execute_query(sql,val)

	message = '''
			A Design Request has been Deleted.<br><br>

			RequestID: ''' + removeid + '''<br>
			Reason for deletion: ''' + delcomment + '''<br>
			Deleted by: '''+ deleted_by[0][0]+'''<br><br>

			 Thanks,<br>ERAM.'''
	subject = "[Request ID :"+removeid+"] Design Review Request Deleted"

	for email in admin_email:
		send_mail(email[0], subject, message)

	for i in range(len(eids)):
		if(eids[i][0]):
			send_mail(eids[i][0], subject, message)

	return redirect(url_for('request_review', _external=True))

@app.route('/update_req',methods = ['POST', 'GET'])			##method for requestor to delete board request
def update_req():
	usernames = request.form.getlist("usernames")

	upid=request.form.get("updateid")

	cad=request.form.get("cadlead")
	deslead=request.form.get("designlead")
	manager=request.form.get("des_manager")
	start=request.form.get("start")
	tapeout=request.form.get("end")
	track=request.form.get("track"+upid)
	comment=request.form.get("comments"+upid)
	user = session.get('username')
	board_id = 0

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	sql = "SELECT DesignLeadWWID,DesignManagerWWID,CADLeadWWID,StartDate,EndDate,ReviewTimelineID,Comments FROM BoardDetailsRequest WHERE RequestID= %s"
	val=(upid,)
	request_details = execute_query(sql,val)


	sql = "SELECT BoardTrackID FROM BoardTrack WHERE BoardTrackName= %s"
	val=(track,)
	boardtrack = execute_query(sql,val)
	eids=[]
	if (usernames):
		sql = "select EmailID from HomeTable where UserName in %s"
		val = (usernames,)
		eids = execute_query(sql, val)

	sql = "SELECT BoardID FROM RequestMap WHERE RequestID = %s"
	val = (upid,)
	bid = execute_query(sql,val)

	# to update last updated time
	sql="UPDATE BoardDetailsRequest set UpdatedOn = %s where RequestID=%s"
	val=(t,upid)
	execute_query(sql,val)

	if(bid != ()):
		sql="UPDATE BoardDetails set UpdatedOn = %s where BoardID=%s"
		val=(t,bid[0][0])
		execute_query(sql,val)


	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val = ('yes',)
	admin_email = execute_query(sql,val)

	updated = 'no'

	subject = "[Request ID: "+upid+"] Design Review Request Updated"

	message0 =" <b>RequestID: </b>" + upid + "<br>"

	if(bid!= ()):
		sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
		val = (bid[0][0],)
		bname = execute_query(sql,val)
		board_id = bid[0][0]

		if(bname != ()):
			message0 =" <b>RequestID: </b>" + upid + "<br><br>" + " <b>Board Name: </b>" + bname[0][0] + "<br>"

	message1=""
	message2=""
	message3=""
	message4=""
	message5=""
	message6=""
	message7=""
	message8 = "A Design Request has been Updated by <b>" + user + "</b>. <br><br>"
	message9 = "<br>Thanks,<br>ERAM."

	if(cad):

		sql="select WWID from HomeTable where UserName=%s"
		val=(cad,)
		cadid=execute_query(sql,val)

		if cadid != ():

			sql = "SELECT CADLeadWWID FROM BoardDetailsRequest WHERE RequestID = %s"
			val = (upid,)
			cad_data = execute_query(sql,val)

			if(cad_data != ()):

				if (cad_data[0][0] != cadid[0][0]):

					updated  ='yes'

					sql="Update BoardDetailsRequest set CADLeadWWID = %s where RequestID=%s"
					val=(cadid[0][0],upid)
					execute_query(sql,val)
					if(bid != ()):
						sql="Update BoardDetails set CADLeadWWID = %s where BoardID=%s"
						val=(cadid[0][0],bid[0][0])
						execute_query(sql,val)

					message1 = '''
								<br>

								<b>Layout Lead: </b>''' + cad + '''<br>

								 '''

	if(deslead):
		
		sql="select WWID from HomeTable where UserName=%s"
		val=(deslead,)
		desid=execute_query(sql,val)

		if desid != ():

			sql = "SELECT DesignLeadWWID FROM BoardDetailsRequest WHERE RequestID = %s"
			val = (upid,)
			des_data = execute_query(sql,val)

			if(des_data != ()):

				if (des_data[0][0] != desid[0][0]):

					updated = 'yes'

					sql="Update BoardDetailsRequest set DesignLeadWWID = %s where RequestID=%s"
					val=(desid[0][0],upid)
					execute_query(sql,val)

					if(bid != ()):
						sql="Update BoardDetails set DesignLeadWWID = %s where BoardID=%s"
						val=(desid[0][0],bid[0][0])
						execute_query(sql,val)

					message2 = '''
								<br>

						
								<b>Design Lead: </b>''' + deslead + '''<br>

								 '''

	if(manager):

		sql="select WWID from HomeTable where UserName=%s"
		val=(manager,)
		valid_manager_name=execute_query(sql,val)
		
		if valid_manager_name != ():
			sql = "SELECT DesignManagerWWID FROM BoardDetailsRequest WHERE RequestID = %s"
			val = (upid,)
			des_manager_data = execute_query(sql,val)

			if(des_manager_data != ()):

				if (des_manager_data[0][0] != manager):

					updated = 'yes'

					sql="Update BoardDetailsRequest set DesignManagerWWID = %s where RequestID=%s"
					val=(manager,upid)
					execute_query(sql,val)

					if(bid!= ()):
						sql="Update BoardDetails set DesignManagerWWID = %s where BoardID=%s"
						val=(manager,bid[0][0])
						execute_query(sql,val)			

					message3 = '''
									<br>

									
									<b>Design Manager: </b>''' + manager + '''<br>

									 '''

	if(start != str(request_details[0][3])):
		updated = 'yes'
		sql="UPDATE BoardDetailsRequest SET StartDate = %s WHERE RequestID=%s"
		val=(start,upid)
		execute_query(sql,val)	

		if(bid!= ()):


			sql="SELECT StartDate FROM BoardDetailsRequest WHERE RequestID=%s"
			val=(upid,)
			new_start_date=execute_query(sql,val)[0][0]

			sql="SELECT ProposedStartDate,ProposedEndDate FROM DesignCalendar WHERE BoardID=%s"
			val=(bid[0][0],)
			dsdate=execute_query(sql,val)

			if dsdate != ():

				diff_days = (dsdate[0][1] - dsdate[0][0]).days

				diff_ww_days = 0

				for i in range(0,diff_days):
					ww_day_number = get_isocalendar(dsdate[0][0] + datetime.timedelta(days=i))[2]

					if ww_day_number not in (6,7):
						diff_ww_days += 1

				final_end_date = new_start_date + datetime.timedelta(days=0)

				for i in range(0,diff_ww_days):
					valid_ww_day = False
					while not valid_ww_day:
						final_end_date += datetime.timedelta(days=1)
						if get_isocalendar(final_end_date)[2] not in (6,7):
							valid_ww_day = True
						else:
							print("weekend.")

				sql="UPDATE DesignCalendar SET ProposedStartDate = %s,ProposedEndDate = %s, BoardState = 'Design Team Projection' WHERE BoardID=%s"
				val=(start,final_end_date,bid[0][0])
				execute_query(sql,val)	

		message4 = '''
						<br>

						
						<b>Start Date: </b>''' + get_work_week_str_fmt(start) + '''<br>

						 '''
	if(tapeout != str(request_details[0][4])):
		updated = 'yes'
		sql="UPDATE BoardDetailsRequest SET EndDate = %s WHERE RequestID=%s"
		val=(tapeout,upid)
		execute_query(sql,val)

		message5 = '''
							<br>

							
							<b>Tapeout Date: </b>''' + get_work_week_str_fmt(tapeout) + '''<br>

							 '''

	if (track):
		updated = 'yes'
		sql="select Comments from BoardDetailsRequest where RequestID=%s"
		val=(upid,)
		commts=execute_query(sql,val)

		appcomment = user + ": " + comment + ",Last Comment:" + commts[0][0]

		sql = "Update BoardDetailsRequest set BoardTrackID = %s,Comments=%s where RequestID=%s"
		val = (boardtrack,appcomment,upid)
		execute_query(sql, val)

		if track == "Yes":
			fast_track_content = '<span style="font-weight:bold;color:red;">Yes</span>'
		else:
			fast_track_content = 'No'

		message6 = '''
								<br>

								
								<b>FastTrack: </b>''' + fast_track_content + '''<br><br>
								</b>Comments: </b>'''+comment +'''<br>

								 '''
	if ((track=="" or track==None) and comment):
		updated = 'yes'
		sql = "select Comments from BoardDetailsRequest where RequestID=%s"
		val = (upid,)
		commts = execute_query(sql, val)

		appcomment = user + ": " + comment + ",Last Comment:" + commts[0][0]

		sql = "Update BoardDetailsRequest set Comments = %s where RequestID=%s"
		val = (appcomment, upid,)
		execute_query(sql, val)
		message7 = '''
								<br>

								
								<b>Comments: </b>''' + comment + '''<br>

								 '''

	email_list = []
	message=message8+message0+message1+message2+message3+message4+message5+message6+message7+message9
	if(updated == 'yes'):
		for email in admin_email:
			email_list.append(email[0])

		for i in range(len(eids)):
			if(eids[i][0]):
				email_list.append(eids[i][0])


	sql = "SELECT IFNULL(b.EmailID,''),IFNULL(c.EmailID,'') FROM BoardDetailsRequest a LEFT JOIN HomeTable b ON a.CADLeadWWID = b.WWID LEFT JOIN HomeTable c ON a.DesignLeadWWID = c.WWID WHERE a.RequestID = %s"
	val = (upid,)
	rs_leads_data = execute_query(sql,val)

	if rs_leads_data != ():

		if rs_leads_data[0][0] != '':
			email_list.append(rs_leads_data[0][0])

		if rs_leads_data[0][1] != '':
			email_list.append(rs_leads_data[0][1])

	email_list = sorted(set(email_list), reverse=True)
	for i in email_list:
		send_mail(i, subject, message,email_list)

	# log table
	try:
		log_notes = 'User has Updated Board details Request.<br>'+message0+message1+message2+message3+message4+message5+message6+message7
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Design Request',board_id,upid,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	return redirect(url_for('request_review', _external=True))

@app.route("/reopen",methods = ['POST', 'GET'])
def reopen():
	boardid = request.form.get("boardid")
	componentid = request.form.get("componentid")
	remarks = request.form.get("remarks")
	wwid = session.get('wwid')
	name = session.get('username')

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	BoardName = execute_query(sql,val)[0][0]

	sql = "SELECT ComponentName FROM ComponentType WHERE ComponentID = %s"
	val = (componentid,)
	ComponentName = execute_query(sql,val)[0][0]

	# log table
	try:
		log_notes = 'User has Re-opened Interface '+str(ComponentName)+' for Design ID: '+str(boardid)+'<br>Comments: '+str(remarks)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Feedbacks',boardid,0,componentid,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	sql = "UPDATE BoardReview SET SignedOff_Reviewer2 = %s WHERE ComponentID = %s AND BoardID = %s"
	val = ('no',componentid,boardid)
	execute_query(sql,val)

	sql = "UPDATE ScheduleTableComponent SET ScheduleStatusID = %s WHERE BoardID = %s AND ComponentID = %s"
	#val = (6,boardid,componentid)
	val = (2,boardid,componentid)
	execute_query(sql,val)

	sql = "SELECT * FROM BoardReview WHERE BoardID = %s AND ComponentID = %s"
	val = (boardid,componentid)
	feedbacks_rs = execute_query(sql,val)

	if feedbacks_rs == ():
		sql = "UPDATE BoardReviewDesigner SET IsPdgElectricalSubmitted = %s WHERE ComponentID = %s AND BoardID = %s"
		val = (None,componentid,boardid)
		execute_query(sql,val)

	sql = "SELECT IFNULL(CommentSignOffInterface,''),IFNULL(b.UserName,'') FROM BoardReviewDesigner a LEFT JOIN HomeTable b ON a.CommentSignOffInterfaceUpdateBy = b.WWID WHERE BoardID = %s AND ComponentID = %s"
	val = (boardid,componentid)
	signoff_comments_data = execute_query(sql,val)

	if signoff_comments_data != ():

		signoff_comments_data_final = '<br>   ' + str(signoff_comments_data[0][0]) + '<br><span style="color:grey; font-size: smaller;">Signed-Off by: ' + str(signoff_comments_data[0][1]) + '</span>'
		signoff_comments_data_final += '<br><br>   ' + remarks + '<br><span style="color:grey; font-size: smaller;">Reopened by: ' + name + '</span><br>'

		sql = "UPDATE BoardReviewDesigner SET CommentSignOffInterface = %s WHERE BoardID = %s AND ComponentID = %s"
		val = (signoff_comments_data_final,boardid,componentid)
		execute_query(sql,val)	

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)
	

	sql = "select distinct SecondaryWWID from ComponentReview C2, BoardReviewDesigner B1, CategoryLeadTable C3,ComponentType C4 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID = %s AND C2.CategoryID = C3.CategoryID AND C3.SKUID = %s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s"
	val = (boardid,componentid,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	sec_ele_wwid = execute_query(sql,val)

	sec_wwid = []
	for i in range(0,len(sec_ele_wwid)):
		ele = sec_ele_wwid[i]
		for j in range(0,len(ele)):
			sec_ele = ele[j][1:-1]
			spl = sec_ele.split(",")
			for k in range(0,len(spl)):
				sec_wwid.append(spl[k])

	sql = "select distinct SecondaryWWID FROM ComponentDesign C2,BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID = %s"
	val = (boardid,componentid,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	sec_des_wwid = execute_query(sql,val)

	des_wwid = []
	for i in range(0,len(sec_des_wwid)):
		des = sec_des_wwid[i]
		for j in range(0,len(des)):
			sec_des = des[j][1:-1]
			spl = sec_des.split(",")
			for k in range(0,len(spl)):
				des_wwid.append(spl[k])

	email_list = []

	sql = "SELECT DISTINCT H1.EmailID,H4.EmailID FROM BoardReviewDesigner B1,  ComponentReview C2, HomeTable H1,HomeTable H4, CategoryLeadTable C3 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID AND C2.CategoryID = C3.CategoryID AND C3.CategoryLeadWWID = H4.WWID AND C3.SKUID = %s AND C3.PlatformID = %s AND C3.MemTypeID =%s AND C3.DesignTypeID = %s "
	val = (boardid,componentid,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	emailids = execute_query(sql,val)

	for j in sec_wwid:
		sql = 'SELECT DISTINCT EmailID FROM HomeTable WHERE WWID =%s'
		val = (j,)
		eid1 = execute_query(sql,val)
		if(eid1 != ()):
			email_list.append(eid1[0][0])

	
	sql = "SELECT DISTINCT H1.EmailID FROM BoardReviewDesigner B1,  ComponentDesign C2, HomeTable H1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.PrimaryWWID = H1.WWID AND C2.MemTypeID =%s AND C2.DesignTypeID = %s"
	val = (boardid,componentid,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	emailids2 = execute_query(sql,val)

	for j in des_wwid:
		sql = 'SELECT EmailID FROM HomeTable WHERE WWID =%s'
		val = (j,)
		eid1 = execute_query(sql,val)
		if(eid1 != ()):
			email_list.append(eid1[0][0])
	
	for k in emailids:
		email_list.append(k[0])
		email_list.append(k[1])

	for k in emailids2:
		email_list.append(k[0])

	sql="SELECT  a.CategoryName,b.EmailID from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2 WHERE cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID = %s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PLatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID ORDER BY cr.ComponentID"
	val=(sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],componentid,sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	catlead=execute_query(sql,val)
	if(catlead != ()):
		for i in catlead:
			email_list.append(i[1])			

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val = ('yes',)
	admin_email = execute_query(sql,val)
	for k in admin_email:
		email_list.append(k[0])

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
	val = (boardid,)
	designlist = execute_query(query,val)
	designlead_list = []
	for i in range(len(designlist)):
		eid = designlist[0][1]
		email_list.append(eid)


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
	val = (boardid,)
	cadlist = execute_query(query,val)
	cadlead_list = []
	for i in range(len(cadlist)):
		eid = cadlist[0][1]
		email_list.append(eid)


	email_list = sorted(set(email_list), reverse=True)
	subject="[ID: "+boardid+"] "+ ComponentName+" - Reopened by the Electrical Team."
	message=''' ERAM Design ID: ''' + boardid +" - "+BoardName +''' <br><br>
				<b>'''+ComponentName + '''</b> - Reopened by <b> ''' + name + '''</b><br><br>
				<b>Comments: </b><br>           ''' + signoff_comments_data_final + '''<br><br>
				Please proceed to visit https://eram.apps1-fm-int.icloud.intel.com/ to view reopened interface. <br><br><br>
					
				Thanks, <br>
				ERAM.'''


	for m in email_list:
		send_mail(m,subject,message,email_list);

	#return redirect(url_for('reviewer', _external=True))
	return redirect(url_for('feedbacks', _external=True))

@app.route("/save_ss",methods = ['POST', 'GET'])
def screenshot():
	data = request.form.get("base64data")

	im = Image.open(BytesIO(base64.b64decode(data)))
	return "taken"

@app.route("/query.html")
def adm_que():

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'adm_que'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	wwid = session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	name = session.get('username')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	sql="select RoleID from HomeTable where WWID=%s "
	val = (wwid,)
	role=execute_query(sql,val)[0][0]
	if (role == 14):
		mgt_access=True
	else:
		mgt_access=False

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	return render("query.html",is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,mgt_access=mgt_access,is_admin=is_admin,name = name,username=username,user_role_name=user_role_name,region_name=region_name)


@app.route("/admin_query",methods = ['POST', 'GET'])
def admin_query():
	query = request.form.get("query")
	a = execute_query_sql(query)
	clist = {}
	clist["result"]=json.dumps(a)
	return jsonify(clist)

@app.route("/check_submit",methods = ['POST', 'GET'])
def check_submit():
	boardid = request.form.get("boardid")
	componentid = request.form.get("componentid")

	sql = "SELECT PDG_Electrical FROM BoardReviewDesigner WHERE BoardID = %s AND ComponentID = %s "
	val = (boardid,componentid)
	pdg = execute_query(sql,val)[0][0]
	pdg_submit = 'no'

	if(pdg == 'Met' or pdg == 'Not Met'):
		pdg_submit = 'yes'

	sql = "SELECT count(*) FROM BoardReview WHERE BoardID = %s AND ComponentID = %s"
	val = (boardid,componentid)
	result = execute_query(sql,val)

	total_issue_count = 0
	if result != ():
		total_issue_count = result[0][0]

	pdg_not_met_valid_submit = False
	if (pdg == "Not Met") and (total_issue_count == 0):
		pdg_not_met_valid_submit = True

	sql = "SELECT Submitted,Submitted_Designer,Submitted_Reviewer2,RiskLevelSubmitted FROM BoardReview WHERE HasChild IS NULL AND BoardID = %s AND ComponentID = %s AND Submitted = %s"
	val = (boardid,componentid,"yes")
	result = execute_query(sql,val)
	submit = 'yes'
	for i in result:
		'''
		if(i[0] != 'yes'):
			
			submit = 'no'
			break
		'''

		if(i[1] != 'yes'):
			submit = 'no'
			break

		
		if(i[2] != 'yes'):
			submit = 'no'
			break
		

		if(i[3] != 'yes'):
			submit = 'no'
			break

	clist = {}
	clist['sub']=submit
	clist['pdg_submit']=pdg_submit
	clist['pdg_not_met_valid_submit']=pdg_not_met_valid_submit
	clist['total_issue_count']=total_issue_count
	clist['open_issue_count']=0
	clist['saved_issue_count']=0
	clist['saved_issue_username']=''
	saved_issue_username_temp = ''

	sql = "SELECT count(*) FROM BoardReview WHERE HasChild IS NULL AND IssueStatus = 'Open' AND BoardID = %s AND ComponentID = %s"
	val = (boardid,componentid)
	result = execute_query(sql,val)
	if result != ():
		clist['open_issue_count'] = result[0][0]

	sql = "SELECT  count(*) FROM BoardReview B1 WHERE B1.HasChild IS NULL AND B1.BoardID = %s AND B1.ComponentID = %s AND (B1.Submitted IS NULL OR B1.Submitted <> %s)"
	val = (boardid,componentid,"yes")
	result_saved = execute_query(sql,val)
	if result_saved != ():
		clist['saved_issue_count'] = result_saved[0][0]

	sql = "SELECT DISTINCT H.UserName FROM BoardReview B1 LEFT JOIN HomeTable H ON B1.WWIDreviewer = H.WWID WHERE B1.HasChild IS NULL AND B1.BoardID = %s AND B1.ComponentID = %s AND (B1.Submitted IS NULL OR B1.Submitted <> %s)"
	val = (boardid,componentid,"yes")
	result_saved_username = execute_query(sql,val)
	if result_saved_username != ():

		for i in range(0,len(result_saved_username)):
			saved_issue_username_temp += result_saved_username[i][0]+'; '

		clist['saved_issue_username']=saved_issue_username_temp


	return jsonify(clist)

@app.route("/download",methods = ['POST', 'GET'])
def download():
	commentid = request.form.get("commentid")
	number = request.form.get("number")
	if(number == '1'):
		sql = "SELECT ReviewerFilename,ReviewerFile FROM FileStorage WHERE CommentID = %s"
		val = (commentid,)
		files = execute_query(sql,val)

		return send_file(BytesIO(files[0][1]),attachment_filename=files[0][0],as_attachment=True)
	else:
		sql = "SELECT DesignerFilename,DesignerFile FROM FileStorage WHERE CommentID = %s"
		val = (commentid,)
		files = execute_query(sql,val)

		return send_file(BytesIO(files[0][1]),attachment_filename=files[0][0],as_attachment=True)

@app.route("/download_board_excel",methods = ['POST', 'GET'])
def download_board_excel():

	boardid = request.form.get("boardid")
	sql = "SELECT BoardFileName, C1.ComponentName, DesignDocument,SignalName,AreaOfIssue,FeedbackSummary,RiskLevel,ReferenceNumber,ImplementationStatus,Comment,IssueStatus,Comment_Reviewer,RiskLevelSignOff,CommentID,ParentCommentID FROM BoardReview B1,ComponentType C1 WHERE BoardID =%s AND ParentCommentID = 0 AND B1.ComponentID = C1.ComponentID"
	val = (boardid,)
	parents = execute_query(sql,val)

	sql = "SELECT BoardFileName, C1.ComponentName, DesignDocument,SignalName,AreaOfIssue,FeedbackSummary,RiskLevel,ReferenceNumber,ImplementationStatus,Comment,IssueStatus,Comment_Reviewer,RiskLevelSignOff,CommentID,ParentCommentID FROM BoardReview B1, ComponentType C1 WHERE BoardID =%s AND ParentCommentID <> 0 AND B1.ComponentID = C1.ComponentID"
	val = (boardid,)
	children = execute_query(sql,val)

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	boardname = execute_query(sql,val)[0][0]
	result = []

	k=0
	for i in parents:
		result.append(i)
		parentid = i[13]
		while(k<len(children) and children[k][14] == parentid):
			result.append(children[k])
			k = k+1

	si = io.StringIO()
	cw = csv.writer(si)
	ls = ["BoardFileName","InterfaceName","DesignDocument","SignalName","AreaOfIssue","FeedbackSummary","RiskLevel","ReferenceNumber","","ImplementationStatus","Comment","","IssueStatus","Comment_Reviewer","RiskLevelSignOff"]

	for i in result:
		if(i != 0 and i[14] == 0 ):
			ls3 = []
			cw.writerow(ls3)
		ls2 = [i[0],i[1],i[2],i[3],i[4],i[5],i[6],i[7],"",i[8],i[9],"",i[10],i[11],i[12]]
		cw.writerow(ls2)
	response = make_response(si.getvalue())
	response.headers['Content-Disposition'] = 'attachment; filename='+boardname+'.csv'
	response.headers["Content-type"] = "text/csv"
	return response

@app.route("/download_feedback_excel_popup",methods = ['POST', 'GET'])
def download_feedback_excel_popup():

	boardid = request.form.get("boardid")

	result = {}

	sql = "SELECT DISTINCT c2.CategoryID,c2.CategoryName FROM BoardReview b1, ComponentType c1, CategoryType c2 WHERE b1.ComponentID = c1.ComponentID AND c1.CategoryID = c2.CategoryID AND b1.BoardID = %s AND b1.RiskLevel IS NOT NULL AND b1.RiskLevel <> '' ORDER BY FIELD(c2.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13)"
	val = (boardid,)
	interface_category = execute_query(sql,val)

	if interface_category != ():
		result['interface_category'] = interface_category
	else:
		result['interface_category'] = []

	sql = "SELECT DISTINCT c1.ComponentID,c1.ComponentName FROM BoardReview b1, ComponentType c1, CategoryType c2 WHERE b1.ComponentID = c1.ComponentID AND c1.CategoryID = c2.CategoryID AND b1.BoardID = %s AND b1.RiskLevel IS NOT NULL AND b1.RiskLevel <> '' ORDER BY FIELD(c2.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), c1.ComponentName"
	val = (boardid,)
	interface_name = execute_query(sql,val)

	if interface_name != ():
		result['interface_name'] = interface_name
	else:
		result['interface_name'] = []

	sql = "SELECT DISTINCT IFNULL(NULLIF(b1.RiskLevel,''),'None') FROM BoardReview b1 WHERE b1.BoardID = %s AND b1.RiskLevel IS NOT NULL AND b1.RiskLevel <> ''"
	val = (boardid,)
	risk_level = execute_query(sql,val)

	if risk_level != ():
		result['risk_level'] = risk_level
	else:
		result['risk_level'] = []

	sql = "SELECT DISTINCT IFNULL(NULLIF(b1.IssueStatus,''),'None') FROM BoardReview b1 WHERE b1.BoardID = %s AND b1.RiskLevel IS NOT NULL AND b1.RiskLevel <> ''"
	val = (boardid,)
	issue_status = execute_query(sql,val)

	if issue_status != ():
		result['issue_status'] = issue_status
	else:
		result['issue_status'] = []

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	boardname = execute_query(sql,val)

	if boardname != ():
		result['boardname'] = boardname[0][0]
	else:
		result['boardname'] = ''

	return jsonify(result)

@app.route("/get_interface_name",methods = ['POST', 'GET'])
def get_interface_name():

	array_body = request.form.get("array_body")
	data = json.loads(array_body)
	boardid = int(data[0])
	interface_category = data[1]

	result = {}

	# to convert string to int, also as per sql in clause
	interface_category_list = []
	for i in range(0,len(interface_category)):
		interface_category_list.append(int(interface_category[i]))

	interface_category_list = tuple(interface_category_list)

	sql = "SELECT DISTINCT c1.ComponentID,c1.ComponentName FROM BoardReview b1, ComponentType c1, CategoryType c2 WHERE b1.ComponentID = c1.ComponentID AND c1.CategoryID = c2.CategoryID AND b1.BoardID = %s AND c1.CategoryID IN %s AND b1.RiskLevel IS NOT NULL AND b1.RiskLevel <> '' ORDER BY FIELD(c2.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), c1.ComponentName"
	val = (boardid,interface_category_list)
	interface_name = execute_query(sql,val)

	result['interface_name'] = []
	if interface_name != ():
		for i in interface_name:
			result['interface_name'].append(i)

	sql = "SELECT DISTINCT IFNULL(NULLIF(b1.RiskLevel,''),'None') FROM BoardReview b1, ComponentType c1, CategoryType c2  WHERE b1.ComponentID = c1.ComponentID AND c1.CategoryID = c2.CategoryID AND b1.BoardID = %s AND c1.CategoryID IN %s AND b1.RiskLevel IS NOT NULL AND b1.RiskLevel <> ''"
	val = (boardid,interface_category_list)
	risk_level = execute_query(sql,val)

	result['risk_level'] = []
	if risk_level != ():
		for i in risk_level:
			result['risk_level'].append(i)

	sql = "SELECT DISTINCT IFNULL(NULLIF(b1.IssueStatus,''),'None') FROM BoardReview b1, ComponentType c1, CategoryType c2  WHERE b1.ComponentID = c1.ComponentID AND c1.CategoryID = c2.CategoryID AND b1.BoardID = %s AND c1.CategoryID IN %s AND b1.RiskLevel IS NOT NULL AND b1.RiskLevel <> ''"
	val = (boardid,interface_category_list)
	issue_status = execute_query(sql,val)

	result['issue_status'] = []
	if issue_status != ():
		for i in issue_status:
			result['issue_status'].append(i)

	return jsonify(result)

@app.route("/get_risk_level",methods = ['POST', 'GET'])
def get_risk_level():

	array_body = request.form.get("array_body")
	data = json.loads(array_body)
	boardid = int(data[0])
	interface_category = data[1]
	interface_name = data[2]

	result = {}

	interface_category_list = []
	#if not all_selected_interface_category:
	for i in range(0,len(interface_category)):
		interface_category_list.append(int(interface_category[i]))

	interface_category_list = tuple(interface_category_list)

	interface_name_list = []
	#if not all_selected_interface_name:
	for i in range(0,len(interface_name)):
		interface_name_list.append(int(interface_name[i]))

	interface_name_list = tuple(interface_name_list)

	# to convert string to int, also as per sql in clause

	sql = "SELECT DISTINCT IFNULL(NULLIF(b1.RiskLevel,''),'None') FROM BoardReview b1, ComponentType c1, CategoryType c2  WHERE b1.ComponentID = c1.ComponentID AND c1.CategoryID = c2.CategoryID AND b1.BoardID = %s AND c1.ComponentID IN %s AND b1.RiskLevel IS NOT NULL AND b1.RiskLevel <> ''"
	val = (boardid,interface_name_list)
	risk_level = execute_query(sql,val)

	result['risk_level'] = []
	if risk_level != ():
		for i in risk_level:
			result['risk_level'].append(i)

	sql = "SELECT DISTINCT IFNULL(NULLIF(b1.IssueStatus,''),'None') FROM BoardReview b1, ComponentType c1, CategoryType c2  WHERE b1.ComponentID = c1.ComponentID AND c1.CategoryID = c2.CategoryID AND b1.BoardID = %s AND c1.ComponentID IN %s AND b1.RiskLevel IS NOT NULL AND b1.RiskLevel <> ''"
	val = (boardid,interface_name_list)
	issue_status = execute_query(sql,val)

	result['issue_status'] = []
	if issue_status != ():
		for i in issue_status:
			result['issue_status'].append(i)

	return jsonify(result)

@app.route("/get_issue_status",methods = ['POST', 'GET'])
def get_issue_status():

	array_body = request.form.get("array_body")
	data = json.loads(array_body)
	boardid = int(data[0])
	interface_category = data[1]
	interface_name = data[2]
	risk_level = data[3]

	result = {}

	interface_category_list = []
	#if not all_selected_interface_category:
	for i in range(0,len(interface_category)):
		interface_category_list.append(int(interface_category[i]))

	interface_category_list = tuple(interface_category_list)

	interface_name_list = []
	#if not all_selected_interface_name:
	for i in range(0,len(interface_name)):
		interface_name_list.append(int(interface_name[i]))

	interface_name_list = tuple(interface_name_list)

	risk_level_list = []
	#if not all_selected_risk_level:
	for i in range(0,len(risk_level)):
		risk_level_list.append(risk_level[i])

	risk_level_list = tuple(risk_level_list)

	# to convert string to int, also as per sql in clause

	sql = "SELECT DISTINCT IFNULL(NULLIF(b1.IssueStatus,''),'None') FROM BoardReview b1, ComponentType c1, CategoryType c2  WHERE b1.ComponentID = c1.ComponentID AND c1.CategoryID = c2.CategoryID AND b1.BoardID = %s AND c1.ComponentID IN %s AND b1.RiskLevel IN %s AND b1.RiskLevel IS NOT NULL AND b1.RiskLevel <> ''"
	val = (boardid,interface_name_list,risk_level_list)
	issue_status = execute_query(sql,val)

	result['issue_status'] = []
	if issue_status != ():
		for i in issue_status:
			result['issue_status'].append(i)

	return jsonify(result)

@app.route("/download_feedback_excel",methods = ['POST', 'GET'])
def download_feedback_excel():

	boardid = request.form.get("boardid")
	interface_category = request.form.getlist("interface_category")
	interface_name = request.form.getlist("interface_name")
	risk_level = request.form.getlist("risk_level")
	issue_status = request.form.getlist("issue_status")

	interface_category_list = []
	#if not all_selected_interface_category:
	for i in range(0,len(interface_category)):
		interface_category_list.append(int(interface_category[i]))

	interface_category_list = tuple(interface_category_list)

	interface_name_list = []
	#if not all_selected_interface_name:
	for i in range(0,len(interface_name)):
		interface_name_list.append(int(interface_name[i]))

	interface_name_list = tuple(interface_name_list)

	risk_level_list = []
	#if not all_selected_risk_level:
	for i in range(0,len(risk_level)):
		#if risk_level[i] == 'None':
		#	risk_level[i] = ''
		risk_level_list.append(risk_level[i])

	risk_level_list = tuple(risk_level_list)

	issue_status_list = []
	#if not all_selected_issue_status:
	for i in range(0,len(issue_status)):
		if issue_status[i] == 'None':
			issue_status[i] = ''
		issue_status_list.append(issue_status[i])

	issue_status_list = tuple(issue_status_list)

	# Initialise
	Interface_Name = ''
	Primary_Electrical_Owner = ''
	Feedback_No = ''
	Design_Document_Name = ''
	Design_Document_Type = ''
	Signal_Name = ''
	Area_Of_Issue = ''
	Feedback_Summary = ''
	Risk_Level = ''
	Feedback_Reference = ''
	Attachment_Electrical = ''
	Implementation_Status = ''
	Comment = ''
	Attachment_Design = ''
	Issue_Status = ''
	Sign_Off_Comment = ''
	Risk_Level_During_Sign_Off = ''

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	boardname = execute_query(sql,val)

	if boardname != ():
		boardname = boardname[0][0]
	else:
		boardname = 'board_name'

	si = io.StringIO()
	cw = csv.writer(si)

	ls = ['','','','To be filled by Electrical owner','','','','','','','','To be filled by Design owner','','','Sign-Off details by Electrical owner']
	cw.writerow(ls)

	ls = ['Interface Name','Primary Electrical Owner','Feedback No','Design Document Name','Design Document Type','Signal Name','Area Of Issue','Feedback Summary','Risk Level','Feedback Reference','Attachment','Implementation Status','Comment','Attachment','Issue Status','Sign-Off Comment','Risk Level During Sign-Off']
	cw.writerow(ls)

	sql = "SELECT C1.ComponentName,B1.BoardFileName,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ReferenceNumber,B1.ImplementationStatus,B1.Comment,B1.IssueStatus,B1.Comment_Reviewer,B1.RiskLevelSignOff,B1.CommentID,B1.HasChild,B1.ComponentID FROM BoardReview B1, ComponentType C1, CategoryType C2 WHERE C1.ComponentID = B1.ComponentID AND C2.CategoryID = C1.CategoryID "
	sql += " AND B1.BoardID = %s " % (boardid)

	if len(interface_category_list) == 1:
		sql += " AND C1.CategoryID = %s " % (interface_category_list)
	else:
		sql += " AND C1.CategoryID IN %s " % (str(interface_category_list,))

	if len(interface_name_list) == 1:
		sql += " AND B1.ComponentID = %s " % (interface_name_list)
	else:
		sql += " AND B1.ComponentID IN %s " % (str(interface_name_list,))

	if len(risk_level_list) == 1:
		if '' in risk_level_list:
			sql += " AND (B1.RiskLevel = '' OR B1.RiskLevel IS NULL) "
		else:
			sql += " AND B1.RiskLevel = '%s' " % str(risk_level_list[0])
	else:
		if '' in risk_level_list:
			sql += " AND (B1.RiskLevel IS NULL OR B1.RiskLevel IN %s )" % (str(risk_level_list,))
		else:
			sql += " AND B1.RiskLevel IN %s " % (str(risk_level_list,))

	if len(issue_status_list) == 1:
		if '' in issue_status_list:
			sql += " AND (B1.IssueStatus = '' OR B1.IssueStatus IS NULL) "
		else:
			sql += " AND B1.IssueStatus = '%s' " % str(issue_status_list[0])
	else:
		if '' in issue_status_list:
			sql += " AND (B1.IssueStatus IS NULL OR B1.IssueStatus IN %s )" % (str(issue_status_list,))
		else:
			sql += " AND B1.IssueStatus IN %s " % (str(issue_status_list,))

	sql += " AND B1.Submitted = 'yes' AND B1.ParentCommentID = 0 ORDER BY FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName, B1.CommentID ASC "

	result_set = execute_query_sql(sql)

	feedback_number = 1

	for i in range(0,len(result_set)):

		# set proper value

		# increase feedback number for next feedback
		if i > 0:
			if result_set[i][0] == result_set[i-1][0]:
				feedback_number += 1
			else:
				feedback_number = 1

		cw = excel_write(cw=cw,i=i,result_set=result_set,feedback_number = feedback_number,boardid=boardid)

		# check for child feedbacks

		# has child feedback check
		if result_set[i][14] != None:
			temp = "B1.ParentCommentID = " + str(result_set[i][13])
			child_sql = sql.replace("B1.ParentCommentID = 0",temp)

			child_result_set = execute_query_sql(child_sql)

			for j in range(0,len(child_result_set)):
				cw = excel_write(cw=cw,i=j,result_set=child_result_set,feedback_number = feedback_number,boardid=boardid)


	response = make_response(si.getvalue())
	response.headers['Content-Disposition'] = 'attachment; filename='+boardname+'.csv'
	response.headers["Content-type"] = "text/csv"
	return response

def excel_write(cw,i,result_set,feedback_number,boardid):

	Interface_Name = result_set[i][0]

	Primary_Electrical_Owner = ''

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	if sku_plat != ():
		sql = "SELECT H.UserName FROM HomeTable H, ComponentReview C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H.WWID ORDER BY C2.ComponentID"
		val = (result_set[i][15],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		primary_rev = execute_query(sql,val)

		if primary_rev != ():
			Primary_Electrical_Owner = primary_rev[0][0]

	Feedback_No = str(feedback_number)
	Design_Document_Name = result_set[i][1]
	Design_Document_Type = result_set[i][2]

	# to avoid #NAME? issue in excel
	Signal_Name = ''
	if result_set[i][3]:
		if result_set[i][3][0] in ['+','-','*','/','%','=']:
			Signal_Name = ' ' + str(result_set[i][3])
		else:
			Signal_Name = result_set[i][3]

	Area_Of_Issue = result_set[i][4]
	Feedback_Summary = result_set[i][5]
	Risk_Level = result_set[i][6]
	Feedback_Reference = result_set[i][7]
	Attachment_Electrical = 'No'
	Attachment_Design = 'No'

	sql_fn = "SELECT ReviewerFilename,DesignerFilename FROM FileStorage WHERE CommentID = %s"
	val = (result_set[i][13],)
	Filename = execute_query(sql_fn,val)
	if Filename != ():
		if Filename[0][0]:
			url = url_for('download_for_excel', _external=True) + '?commentid= ' + str(result_set[i][13]) + '&number=1'
			Attachment_Electrical = '=HYPERLINK("' + url + '","Download Attachment")'

		if Filename[0][1]:
			url = url_for('download_for_excel', _external=True) + '?commentid= ' + str(result_set[i][13]) + '&number=2'
			Attachment_Design = '=HYPERLINK("' + url + '","Download Attachment")'


	Implementation_Status = result_set[i][8]
	Comment = result_set[i][9]
	Issue_Status = result_set[i][10]

	if result_set[i][11]:
		Sign_Off_Comment = result_set[i][11].replace("<br>---------<br>","\n")
	else:
		Sign_Off_Comment = ''

	Risk_Level_During_Sign_Off = result_set[i][12]

	ls = [Interface_Name,Primary_Electrical_Owner,Feedback_No,Design_Document_Name,Design_Document_Type,Signal_Name,Area_Of_Issue,Feedback_Summary,Risk_Level,Feedback_Reference,Attachment_Electrical,Implementation_Status,Comment,Attachment_Design,Issue_Status,Sign_Off_Comment,Risk_Level_During_Sign_Off]
	cw.writerow(ls)

	return cw
@app.route("/download_for_excel",methods = ['POST', 'GET'])
def download_for_excel():
	commentid = request.args.get("commentid")
	number = request.args.get("number")
	#print("dsfsd",commentid,number)
	if(number == '1'):
		sql = "SELECT ReviewerFilename,ReviewerFile FROM FileStorage WHERE CommentID = %s"
		val = (commentid,)
		files = execute_query(sql,val)

		#print(files)

		return send_file(BytesIO(files[0][1]),attachment_filename=files[0][0],as_attachment=True)
	else:
		sql = "SELECT DesignerFilename,DesignerFile FROM FileStorage WHERE CommentID = %s"
		val = (commentid,)
		files = execute_query(sql,val)

		#print(files)

		return send_file(BytesIO(files[0][1]),attachment_filename=files[0][0],as_attachment=True)

@app.route("/download_board_files",methods = ['POST', 'GET'])
def download_board_files():
	boardid = request.form.get("boardid")

	sql="SELECT a.FileName,b.Files FROM UploadDesignFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.FileID WHERE a.BoardID=%s"
	val = (boardid,)
	files = execute_query(sql,val)

	return send_file(BytesIO(files[0][1]),attachment_filename=files[0][0],as_attachment=True)

@app.route("/download_board_files_new",methods = ['POST', 'GET'])
def download_board_files_new():
	boardid = request.form.get("boardid")
	number = request.form.get("number")

	if number == '1':
		sql="SELECT a.BoardFileName,b.Files FROM UploadDesignFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.BoardFileID WHERE a.BoardID=%s"
		val = (boardid,)
		files = execute_query(sql,val)

	elif number == '2':
		sql="SELECT a.SchematicsName,b.Files FROM UploadDesignFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.SchematicsFileID WHERE a.BoardID=%s"
		val = (boardid,)
		files = execute_query(sql,val)

	elif number == '3':
		sql="SELECT a.StackupFileName,b.Files FROM UploadDesignFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.StackupFileID WHERE a.BoardID=%s"
		val = (boardid,)
		files = execute_query(sql,val)

	elif number == '4':
		sql="SELECT a.LengthReportFileName,b.Files FROM UploadDesignFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.LengthReportFileID WHERE a.BoardID=%s"
		val = (boardid,)
		files = execute_query(sql,val)

	elif number == '5':
		sql="SELECT a.OthersFileName,b.Files FROM UploadDesignFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.OthersFileID WHERE a.BoardID=%s"
		val = (boardid,)
		files = execute_query(sql,val)

	return send_file(BytesIO(files[0][1]),attachment_filename=files[0][0],as_attachment=True)

@app.route("/download_signoff_files_latest",methods = ['POST', 'GET'])
def download_signoff_files_latest():
	boardid = request.form.get("boardid")
	number = request.form.get("number")

	if number == '1':
		sql="SELECT a.BoardFileName,b.Files FROM UploadSignoffLatestFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.BoardFileID WHERE a.BoardID=%s"
		val = (boardid,)
		files = execute_query(sql,val)

	elif number == '2':
		sql="SELECT a.SchematicsName,b.Files FROM UploadSignoffLatestFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.SchematicsFileID WHERE a.BoardID=%s"
		val = (boardid,)
		files = execute_query(sql,val)

	elif number == '3':
		sql="SELECT a.StackupFileName,b.Files FROM UploadSignoffLatestFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.StackupFileID WHERE a.BoardID=%s"
		val = (boardid,)
		files = execute_query(sql,val)

	elif number == '4':
		sql="SELECT a.LengthReportFileName,b.Files FROM UploadSignoffLatestFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.LengthReportFileID WHERE a.BoardID=%s"
		val = (boardid,)
		files = execute_query(sql,val)

	elif number == '5':
		sql="SELECT a.OthersFileName,b.Files FROM UploadSignoffLatestFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.OthersFileID WHERE a.BoardID=%s"
		val = (boardid,)
		files = execute_query(sql,val)

	elif number == '6':
		sql="SELECT a.FileName,b.Files FROM UploadSignOffFiles a LEFT JOIN UploadFileStorage b ON b.FileID=a.FileID WHERE a.BoardID=%s ORDER BY a.Insert_Time DESC LIMIT 1"
		val = (boardid,)
		files = execute_query(sql,val)

	return send_file(BytesIO(files[0][1]),attachment_filename=files[0][0],as_attachment=True)

@app.route("/download_comp",methods = ['POST', 'GET'])
def download_comp():
	boardid = request.form.get("boardid")
	componentid = request.form.get("componentid")

	sql = "SELECT a.FileName,b.Files FROM UploadSignOffFiles a LEFT JOIN UploadFileStorage b ON a.BoardID=b.BoardID AND a.FileID=b.FileID WHERE a.BoardID = %s AND a.ComponentID = %s "
	val = (boardid,componentid)
	files = execute_query(sql,val)

	return send_file(BytesIO(files[0][1]),attachment_filename=files[0][0],as_attachment=True)

@app.route("/download_comp_files",methods = ['POST', 'GET'])
def download_comp_files():
	boardid = request.form.get("boardid")
	componentid = request.form.get("componentid")
	number = str(request.form.get("number"))

	files = []

	if number == '1':
		sql = "SELECT a.BoardFileName,b.Files FROM UploadSignOffFiles a LEFT JOIN UploadFileStorage b ON a.BoardID=b.BoardID AND a.BoardFileID=b.FileID WHERE a.BoardID = %s AND a.ComponentID = %s "
		val = (boardid,componentid)
		files = execute_query(sql,val)

	if number == '2':
		sql = "SELECT a.SchematicsName,b.Files FROM UploadSignOffFiles a LEFT JOIN UploadFileStorage b ON a.BoardID=b.BoardID AND a.SchematicsFileID=b.FileID WHERE a.BoardID = %s AND a.ComponentID = %s "
		val = (boardid,componentid)
		files = execute_query(sql,val)

	if number == '3':
		sql = "SELECT a.StackupFileName,b.Files FROM UploadSignOffFiles a LEFT JOIN UploadFileStorage b ON a.BoardID=b.BoardID AND a.StackupFileID=b.FileID WHERE a.BoardID = %s AND a.ComponentID = %s "
		val = (boardid,componentid)
		files = execute_query(sql,val)

	if number == '4':
		sql = "SELECT a.LengthReportFileName,b.Files FROM UploadSignOffFiles a LEFT JOIN UploadFileStorage b ON a.BoardID=b.BoardID AND a.LengthReportFileID=b.FileID WHERE a.BoardID = %s AND a.ComponentID = %s "
		val = (boardid,componentid)
		files = execute_query(sql,val)

	if number == '5':
		sql = "SELECT a.OthersFileName,b.Files FROM UploadSignOffFiles a LEFT JOIN UploadFileStorage b ON a.BoardID=b.BoardID AND a.OthersFileID=b.FileID WHERE a.BoardID = %s AND a.ComponentID = %s "
		val = (boardid,componentid)
		files = execute_query(sql,val)

	if number == '6':
		sql = "SELECT a.FileName,b.Files FROM UploadSignOffFiles a LEFT JOIN UploadFileStorage b ON a.BoardID=b.BoardID AND a.FileID=b.FileID WHERE a.BoardID = %s AND a.ComponentID = %s "
		val = (boardid,componentid)
		files = execute_query(sql,val)

	if files != ():
		if files[0][0] is not None:
			return send_file(BytesIO(files[0][1]),attachment_filename=files[0][0],as_attachment=True)

	return True

@app.route("/check_latest",methods = ['POST', 'GET'])
def check_latest():
	boardid = request.form.get("boardid")
	componentid = request.form.get("componentid")

	clist = {}

	#sql = "SELECT Insert_Time FROM UploadSignOffFiles WHERE BoardID = %s AND ComponentID = %s"
	sql = "SELECT Insert_Time FROM UploadSignOffFilesTemp WHERE BoardID = %s AND ComponentID = %s"
	val = (boardid,componentid)
	rs_t1 = execute_query(sql,val)

	if rs_t1 != ():
		t1 = rs_t1[0][0]
	else:
		clist['proceed'] = 'no'
		return jsonify(clist)

	sql = "SELECT UpdatedOnForDesignSection FROM BoardReview WHERE BoardID =%s AND ComponentID = %s AND Submitted = 'yes' AND (Submitted_Designer IS NULL OR Submitted_Designer <> 'yes') ORDER BY CommentID DESC LIMIT 0,1"
	val = (boardid,componentid)
	feed = execute_query(sql,val)

	if(feed == ()):
		#sql = "SELECT Count FROM UploadSignOffFiles WHERE BoardID = %s AND ComponentID = %s"
		sql = "SELECT Count FROM UploadSignOffFilesTemp WHERE BoardID = %s AND ComponentID = %s"
		val = (boardid,componentid)
		count = execute_query(sql,val)[0][0]

		if(count == 1):
			clist['proceed'] = 'yes'
		else:
			clist['proceed'] = 'no'
	else:	

		time1 = t1
		time2 = feed[0][0]

		try:
			if(time1 > time2):
				clist['proceed'] = 'yes'
			else:
				clist['proceed'] = 'no'
		except:
			clist['proceed'] = 'no'

	return jsonify(clist)

@app.route("/signoff_reviewer",methods = ['POST', 'GET'])
def sigoff_reviewer():

	valid_wwid = False

	if session.get('wwid'):
		wwid = session.get('wwid')
		username=session.get("username")
		if len(wwid) == 8:
			valid_wwid = True
	
	if not valid_wwid:
		session['target_page'] = 'feedbacks'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url
		return redirect(redirect_url)

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	data = {}

	component = request.form.get("component")
	board = request.form.get("board")
	comment = request.form.get("comment")
	comp_selected_list = eval(request.form.get("comp_select[]"))

	BoardName = ""
	is_rev0p6_design = False
	is_rev1p0_design = True

	sql = "SELECT BoardName,ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (board,)
	BoardNamers = execute_query(sql,val)

	if BoardNamers != ():
		BoardName = BoardNamers[0][0]

		if BoardNamers[0][1] in [1,'1']:
			is_rev0p6_design = True
			is_rev1p0_design = False

	sql = "SELECT ComponentName FROM ComponentType WHERE ComponentID = %s"
	val = (component,)
	ComponentName = execute_query(sql,val)[0][0]

	# log table
	try:
		log_notes = 'User has Signed-Off Interface for Interface: '+str(ComponentName)+' & Design ID: '+str(board)+'<br>Comments: '+str(comment)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Feedbacks',board,0,component,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	saved_issue_count = 0
	saved_issue_username = ''
	saved_issue_username_temp = ''

	sql = "SELECT  count(*) FROM BoardReview B1 WHERE B1.HasChild IS NULL AND B1.BoardID = %s AND B1.ComponentID = %s AND (B1.Submitted IS NULL OR B1.Submitted <> %s)"
	val = (board,component,"yes")
	result_saved = execute_query(sql,val)
	if result_saved != ():
		saved_issue_count = result_saved[0][0]

	sql = "SELECT DISTINCT H.UserName FROM BoardReview B1 LEFT JOIN HomeTable H ON B1.WWIDreviewer = H.WWID WHERE B1.HasChild IS NULL AND B1.BoardID = %s AND B1.ComponentID = %s AND (B1.Submitted IS NULL OR B1.Submitted <> %s)"
	val = (board,component,"yes")
	result_saved_username = execute_query(sql,val)
	if result_saved_username != ():

		for i in range(0,len(result_saved_username)):
			saved_issue_username_temp += result_saved_username[i][0]+'; '

		saved_issue_username=saved_issue_username_temp

	# delete any saved feedbacks
	sql = "DELETE FROM BoardReview WHERE HasChild IS NULL AND BoardID = %s AND ComponentID = %s AND (Submitted IS NULL OR Submitted <> %s)"
	val = (board,component,"yes")
	execute_query(sql,val)
	
	sql = "UPDATE BoardReview SET SignedOff_Reviewer2 = %s, Submitted_Reviewer2 = %s WHERE ComponentID = %s AND BoardID = %s"
	val = ('yes','yes',component,board)
	execute_query(sql,val)

	sql = "UPDATE BoardReviewDesigner SET CommentSignOffInterface = CONCAT(CommentSignOffInterface,%s), CommentSignOffInterfaceUpdateBy = %s, IsPdgElectricalSubmitted = %s WHERE ComponentID = %s AND BoardID = %s"
	val = (comment,wwid,'yes',component,board)
	execute_query(sql,val)

	sql = "UPDATE ScheduleTableComponent SET ScheduleStatusID = %s WHERE BoardID = %s AND ComponentID = %s"
	val = (1,board,component)
	execute_query(sql,val)

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (board,)
	sku_plat = execute_query(sql,val)

	if saved_issue_count > 0:
		comment += '<br><br>There are '+str(saved_issue_count)+' saved feedbacks by  user '+saved_issue_username+' are deleted by '+username+'.'
			
	#getting the secondary wwid's from the componentreview from database
	email_list = []
	sql = "select distinct SecondaryWWID from ComponentReview C2, BoardReviewDesigner B1, CategoryLeadTable C3,ComponentType C4 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.CategoryID = C3.CategoryID AND C3.SKUID = %s AND C3.PlatformID = %s  AND C3.MemTypeID = %s AND C3.DesignTypeID = %s"
	val = (board,component,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	sec_ele_wwid = execute_query(sql,val)

	sec_wwid = []
	for i in range(0,len(sec_ele_wwid)):
		ele = sec_ele_wwid[i]
		for j in range(0,len(ele)):
			sec_ele = ele[j][1:-1]
			spl = sec_ele.split(",")
			for k in range(0,len(spl)):
				sec_wwid.append(spl[k])

	sql = "select distinct SecondaryWWID FROM ComponentDesign C2,BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND C2.MemTypeID = %s AND C2.DesignTypeID = %s"
	val = (board,component,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	sec_des_wwid = execute_query(sql,val)

	des_wwid = []
	for i in range(0,len(sec_des_wwid)):
		des = sec_des_wwid[i]
		for j in range(0,len(des)):
			sec_des = des[j][1:-1]
			spl = sec_des.split(",")
			for k in range(0,len(spl)):
				des_wwid.append(spl[k])

	email_list = []

	sql = "SELECT DISTINCT H1.EmailID,H4.EmailID FROM BoardReviewDesigner B1,  ComponentReview C2, HomeTable H1, HomeTable H2,HomeTable H4, CategoryLeadTable C3 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID AND C2.CategoryID = C3.CategoryID AND C3.CategoryLeadWWID = H4.WWID AND C3.SKUID = %s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s "
	val = (board,component,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	emailids = execute_query(sql,val)

	for j in sec_wwid:
		sql = 'SELECT DISTINCT EmailID FROM HomeTable WHERE WWID =%s'
		val = (j,)
		eid1 = execute_query(sql,val)
		if(eid1 != ()):
			email_list.append(eid1[0][0])
	
	sql = "SELECT DISTINCT H1.EmailID FROM BoardReviewDesigner B1,  ComponentDesign C2, HomeTable H1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
	val = (board,component,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	emailids2 = execute_query(sql,val)

	for j in des_wwid:
		sql = 'SELECT EmailID FROM HomeTable WHERE WWID =%s'
		val = (j,)
		eid1 = execute_query(sql,val)
		if(eid1 != ()):
			email_list.append(eid1[0][0])

	name = session.get('username')
	for k in emailids:
		email_list.append(k[0])
		email_list.append(k[1])
	
	for k in emailids2:
		email_list.append(k[0])
			
	sql="SELECT  a.CategoryName,b.EmailID from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2 WHERE cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID = %s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PLatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID ORDER BY cr.ComponentID"
	val=(sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],component,sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	catlead=execute_query(sql,val)
	if(catlead != ()):
		for i in catlead:
			email_list.append(i[1])			

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val = ('yes',)
	admin_email = execute_query(sql,val)
	for k in admin_email:
		email_list.append(k[0])

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
	val = (board,)
	designlist = execute_query(query,val)
	designlead_list = []
	for i in range(len(designlist)):
		eid = designlist[0][1]
		email_list.append(eid)


	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
	val = (board,)
	cadlist = execute_query(query,val)
	cadlead_list = []
	for i in range(len(cadlist)):
		eid = cadlist[0][1]
		email_list.append(eid)


	sql = "SELECT PDG_Electrical FROM BoardReviewDesigner WHERE BoardID = %s AND ComponentID = %s "
	val = (board,component)
	try:
		PDG = execute_query(sql,val)[0][0]	
	except:
		PDG = ''

	email_list = sorted(set(email_list), reverse=True)
	subject="[ID:"+board+"] "+ ComponentName+"- Signed-Off by Electrical Owner"
	message=''' ERAM Design ID: ''' + board +" - "+BoardName +''' <br><br>
			'''+ComponentName + ''' - Signed-Off By : ''' + name + '''<br><br>
				PDG Compliance By Electrical Team: ''' + PDG + ''' <br><br>
				Signed-Off Comments: ''' + comment + '''<br><br>
				Please proceed to visit https://eram.apps1-fm-int.icloud.intel.com/ to view updated feedback. <br><br>

				Thanks, <br>

				ERAM.'''


	for m in email_list:
		send_mail(m,subject,message,email_list)			

	# to notify admin, if all interfaces got signed-off to clode the design
	sql = "SELECT * FROM ScheduleTableComponent a WHERE a.BoardID = %s AND a.ScheduleStatusID in %s AND a.ComponentID IN (SELECT x.ComponentID FROM BoardReviewDesigner x WHERE x.BoardID = %s AND x.IsPdgDesignSubmitted = %s)"
	val = (board,[2,3,6],board,"yes")
	active_interfaces = execute_query(sql,val)

	if active_interfaces == ():
		signoff_2(is_automation=True,boardid=board)

		#subject="[ID: "+board+" - "+BoardName+"] - All Interfaces have been Closed"
		#message='''Hi Admin, <br><br>
		#		<font style="color:green">All Interfaces have been Closed for Design [ID: ''' + board +"] - "+BoardName +'''.<br><br>Please proceed to Sign-Off the Design.</font><br><br>
		#			Thanks, <br>
		#			ERAM.'''
		#email_list = []
		#for k in admin_email:
		#	email_list.append(k[0])
		#for m in email_list:
		#	send_mail(m,subject,message,email_list)			


	board_id = board
	comp_id = component
	# to replace the component details section in ajax

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	is_admin = False
	if has_admin_access == "yes":
		is_admin = True

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	sql = "SELECT * FROM BoardDetails WHERE BoardID = %s AND (DesignLeadWWID = %s OR CADLeadWWID = %s)"	
	val = (board_id,wwid,wwid)
	rs_design_layout_lead = execute_query(sql,val)

	is_design_layout_lead = False
	if rs_design_layout_lead != ():
		is_design_layout_lead = True

	try:
		sql = "SELECT ScheduleStatusID FROM ScheduleTable WHERE BoardID = %s"
		val = (board_id,)
		board_status = execute_query(sql,val)[0][0]
	except:
		board_status = 0

	comp_list = []
	comp_list = get_all_interfaces_feedbacks(boardid=board_id)[1]

	my_designs_id = []
	temp_data = []
	temp_data = get_my_designs_feedbacks()		

	for i in range(0,len(temp_data[1])):
		my_designs_id.append(temp_data[1][i][0])

	my_components_id = []

	temp_data = []

	# for design & Layout Lead, Managers - both My interface and All interface should be same and have edit access, as we dont have any mapping for design/layout lead and managers at backend properly
	if is_design_layout_lead:
		temp_data = get_all_interfaces_feedbacks(boardid=board_id)
	else:
		temp_data = get_my_interfaces_feedbacks(boardid=board_id)


	for i in range(0,len(temp_data[1])):
		my_components_id.append(temp_data[1][i][0])


	data = get_feedbacks_data_page(data=data,boardid=board_id,compid=comp_id,complist=comp_list,sku_plat=sku_plat,board_status=board_status,my_designs_id=my_designs_id,my_components_id=my_components_id)

	sql = "SELECT AreaofIssue FROM AreaOfIssue ORDER BY AreaofIssue"
	area = execute_query_sql(sql)

	areas=[]
	for i in area:
		areas.append(i[0])

	return render('feedbacks_files_div_data.html',is_rev0p6_design=is_rev0p6_design,is_rev1p0_design=is_rev1p0_design,data=data,areas=areas,boardid=board_id,comp_id=comp_id,comp_selected_list=comp_selected_list,is_admin=is_admin,is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner)


@app.route('/data_mining.html',methods = ['POST', 'GET'])
def data_mining():

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'data_mining'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	username=session.get("username")
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	wwid=session.get("wwid")
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "SELECT MIN(ProposedStartDate) FROM DesignCalendar"
	start_date_min = execute_query_sql(sql)[0][0]

	sql = "SELECT MAX(ProposedEndDate) FROM DesignCalendar"
	start_date_max = execute_query_sql(sql)[0][0]

	start_date_default = start_date_min
	end_date_default = start_date_max

	end_date_min = start_date_min
	end_date_max = start_date_max

	start_date_default_ww = str(get_work_week_fun_with_year(start_date_min))
	end_date_default_ww = str(get_work_week_fun_with_year(start_date_max))

	return render("data_mining.html",username=username,user_role_name=user_role_name,region_name=region_name,start_date_default=start_date_default,end_date_default=end_date_default,start_date_min=start_date_min,start_date_max=start_date_max,end_date_min=end_date_min,end_date_max=end_date_max,start_date_default_ww=start_date_default_ww,end_date_default_ww=end_date_default_ww)

@app.route('/data_mining_design_rev',methods = ['POST', 'GET'])
def data_mining_design_rev():
	username=session.get("username")
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	wwid=session.get("wwid")

	start_calender_date = request.form.get("start_calender_date")
	end_calender_date = request.form.get("end_calender_date")

	start_date_default = start_calender_date
	end_date_default = end_calender_date

	sql = "SELECT MIN(ProposedStartDate) FROM DesignCalendar"
	start_date_min = execute_query_sql(sql)[0][0]

	sql = "SELECT MAX(ProposedEndDate) FROM DesignCalendar"
	start_date_max = execute_query_sql(sql)[0][0]

	#end_date_min = start_date_min
	end_date_min = start_date_default
	end_date_max = start_date_max

	start_year = str(datetime.datetime.strptime(start_calender_date, '%Y-%m-%d').year)
	end_year = str(datetime.datetime.strptime(end_calender_date, '%Y-%m-%d').year)

	start_date_default_ww = str(get_work_week_fun_with_year(date_value=datetime.datetime.strptime(start_calender_date, '%Y-%m-%d')))
	end_date_default_ww = str(get_work_week_fun_with_year(date_value=datetime.datetime.strptime(end_calender_date, '%Y-%m-%d')))

	sql="SELECT DISTINCT d1.BoardID,b1.BoardName,s2.ScheduleTypeName,d1.ProposedStartDate,d1.ProposedEndDate FROM DesignCalendar d1,BoardDetails b1,ScheduleTable s1,ScheduleStatusType s2 WHERE b1.BoardID = d1.BoardID AND s1.BoardID = b1.BoardID AND s2.ScheduleID = s1.ScheduleStatusID AND d1.ProposedEndDate >= %s AND d1.ProposedStartDate <= %s order by d1.BoardID desc"
	val=(start_calender_date,end_calender_date)
	result=execute_query(sql,val)

	count_total = 0
	count_signoff = 0
	count_not_signoff = 0
	count_ongoing = 0
	count_yet_to_kickstart = 0
	count_reject = 0

	design_total = []
	design_acive = []
	design_sign_off = []
	design_no_sign_off = []
	design_ongoing = []
	design_yet_to_kickstart = []
	design_reject = []

	if result != ():
		for i in range(0,len(result)):

			count_total += 1

			design_total.append([])
			design_total[i].append(result[i][0])
			design_total[i].append(result[i][1])
			timeline = str(get_work_week_fun_with_year(date_value=result[i][3])) + ' - ' + str(get_work_week_fun_with_year(date_value=result[i][4]))
			design_total[i].append(timeline)

			if result[i][2] == 'Signed-Off':
				count_signoff += 1

				design_acive.append([])
				design_acive[-1].append(result[i][0])
				design_acive[-1].append(result[i][1])
				timeline = str(get_work_week_fun_with_year(date_value=result[i][3])) + ' - ' + str(get_work_week_fun_with_year(date_value=result[i][4]))
				design_acive[-1].append(timeline)

				design_sign_off.append([])
				design_sign_off[-1].append(result[i][0])
				design_sign_off[-1].append(result[i][1])
				timeline = str(get_work_week_fun_with_year(date_value=result[i][3])) + ' - ' + str(get_work_week_fun_with_year(date_value=result[i][4]))
				design_sign_off[-1].append(timeline)

			if result[i][2] == 'No_Signoff':
				count_not_signoff += 1

				design_acive.append([])
				j = -1
				design_acive[j].append(result[i][0])
				design_acive[j].append(result[i][1])
				timeline = str(get_work_week_fun_with_year(date_value=result[i][3])) + ' - ' + str(get_work_week_fun_with_year(date_value=result[i][4]))
				design_acive[j].append(timeline)

				design_no_sign_off.append([])
				j = -1
				design_no_sign_off[j].append(result[i][0])
				design_no_sign_off[j].append(result[i][1])
				timeline = str(get_work_week_fun_with_year(date_value=result[i][3])) + ' - ' + str(get_work_week_fun_with_year(date_value=result[i][4]))
				design_no_sign_off[j].append(timeline)

			if result[i][2] == 'Ongoing':
				count_ongoing += 1

				design_acive.append([])
				j = -1
				design_acive[j].append(result[i][0])
				design_acive[j].append(result[i][1])
				timeline = str(get_work_week_fun_with_year(date_value=result[i][3])) + ' - ' + str(get_work_week_fun_with_year(date_value=result[i][4]))
				design_acive[j].append(timeline)

				design_ongoing.append([])
				j = -1
				design_ongoing[j].append(result[i][0])
				design_ongoing[j].append(result[i][1])
				timeline = str(get_work_week_fun_with_year(date_value=result[i][3])) + ' - ' + str(get_work_week_fun_with_year(date_value=result[i][4]))
				design_ongoing[j].append(timeline)

			if result[i][2] == 'Yet_to_Kickstart':
				count_yet_to_kickstart += 1

				design_acive.append([])
				j = -1
				design_acive[j].append(result[i][0])
				design_acive[j].append(result[i][1])
				timeline = str(get_work_week_fun_with_year(date_value=result[i][3])) + ' - ' + str(get_work_week_fun_with_year(date_value=result[i][4]))
				design_acive[j].append(timeline)

				design_yet_to_kickstart.append([])
				j = -1
				design_yet_to_kickstart[j].append(result[i][0])
				design_yet_to_kickstart[j].append(result[i][1])
				timeline = str(get_work_week_fun_with_year(date_value=result[i][3])) + ' - ' + str(get_work_week_fun_with_year(date_value=result[i][4]))
				design_yet_to_kickstart[j].append(timeline)

			if result[i][2] == 'Rejected':
				count_reject += 1

				design_reject.append([])
				j = -1
				design_reject[j].append(result[i][0])
				design_reject[j].append(result[i][1])
				timeline = str(get_work_week_fun_with_year(date_value=result[i][3])) + ' - ' + str(get_work_week_fun_with_year(date_value=result[i][4]))
				design_reject[j].append(timeline)

	count_reviewed = int(count_signoff + count_not_signoff + count_ongoing + count_yet_to_kickstart)
	count = [count_total,count_signoff,count_not_signoff,count_ongoing,count_yet_to_kickstart,count_reject,count_reviewed]

	return render("data_mining_design_rev.html",user_role_name=user_role_name,region_name=region_name,design_total=design_total,design_acive=design_acive,design_sign_off=design_sign_off,design_no_sign_off=design_no_sign_off,design_ongoing=design_ongoing,design_yet_to_kickstart=design_yet_to_kickstart,design_reject=design_reject,start_year=start_year,count=count,end_year=end_year,username=username,start_date_default=start_date_default,end_date_default=end_date_default,start_date_min=start_date_min,start_date_max=start_date_max,end_date_min=end_date_min,end_date_max=end_date_max,start_date_default_ww=start_date_default_ww,end_date_default_ww=end_date_default_ww)

@app.route('/data_mining_gen_reports',methods = ['POST', 'GET'])
def data_mining_gen_reports():
	#return 'Work in progress...'
	username=session.get("username")
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	wwid=session.get("wwid")
	start_calender_date = request.form.get("start_calender_date")
	end_calender_date = request.form.get("end_calender_date")
	show_report = request.form.get("show_report")

	input_platform = request.form.getlist('platform')
	input_sku = request.form.getlist('sku')
	input_design_type = request.form.getlist('design_type')
	input_review_phase = request.form.getlist('review_phase')
	input_designs = request.form.getlist('designs')

	gen_rep_board = request.form.getlist('designs')

	start_date_default = start_calender_date
	end_date_default = end_calender_date

	sql = "SELECT MIN(ProposedStartDate) FROM DesignCalendar"
	start_date_min = execute_query_sql(sql)[0][0]

	sql = "SELECT MAX(ProposedEndDate) FROM DesignCalendar"
	start_date_max = execute_query_sql(sql)[0][0]

	end_date_min = start_date_default
	end_date_max = start_date_max

	start_year = str(datetime.datetime.strptime(start_calender_date, '%Y-%m-%d').year)
	end_year = str(datetime.datetime.strptime(end_calender_date, '%Y-%m-%d').year)

	start_date_default_ww = str(get_work_week_fun_with_year(date_value=datetime.datetime.strptime(start_calender_date, '%Y-%m-%d')))
	end_date_default_ww = str(get_work_week_fun_with_year(date_value=datetime.datetime.strptime(end_calender_date, '%Y-%m-%d')))

	sql="SELECT DISTINCT d1.BoardID,b1.BoardName,b1.PlatformID,p1.PlatformName,b1.SKUID,s1.SKUName,b1.DesignTypeID,d2.DesignTypeName,b1.ReviewTimelineID,r1.ReviewTimelineName FROM DesignCalendar d1,BoardDetails b1,BoardReview b2,Platform p1,SUK s1,DesignType d2,ReviewTimeline r1 WHERE b1.BoardID = d1.BoardID AND b1.BoardID = b2.BoardID AND p1.PlatformID = b1.PlatformID AND b1.SKUID = s1.SKUID AND b1.DesignTypeID = d2.DesignTypeID AND b1.ReviewTimelineID = r1.ReviewTimelineID AND d1.ProposedEndDate >= %s AND d1.ProposedStartDate <= %s AND b2.DesignDocument <> '' order by d1.BoardID DESC"
	val=(start_calender_date,end_calender_date)
	result=execute_query(sql,val)

	designs = []
	platform = []
	sku = []
	design_type = []
	review_phase = []
	temp_board_id = []

	designs_temp = []
	platform_temp = []
	sku_temp = []
	design_type_temp = []
	review_phase_temp = []

	has_data = False

	if result != ():
		has_data = True

		for i in range(len(result)):
			designs_temp.append([result[i][0],result[i][1],"checked"])
			platform_temp.append([result[i][2],result[i][3],"checked"])
			sku_temp.append([result[i][4],result[i][5],"checked"])
			design_type_temp.append([result[i][6],result[i][7],"checked"])
			review_phase_temp.append([result[i][8],result[i][9],"checked"])

	platform_temp = list(frozenset(tuple(row) for row in platform_temp)) 
	sku_temp = list(frozenset(tuple(row) for row in sku_temp)) 
	design_type_temp = list(frozenset(tuple(row) for row in design_type_temp)) 
	review_phase_temp = list(frozenset(tuple(row) for row in review_phase_temp)) 	

	for row in platform_temp:
		platform.append(list(row))

	for row in sku_temp:
		sku.append(list(row))

	for row in design_type_temp:
		design_type.append(list(row))

	for row in review_phase_temp:
		review_phase.append(list(row))

	for row in designs_temp:
		designs.append(list(row))									

	# sorting
	platform.sort(key = lambda x: x[1])
	sku.sort(key = lambda x: x[1])
	design_type.sort(key = lambda x: x[1])
	review_phase.sort(key = lambda x: x[1])

	temp = 0
	for i in designs:
		temp += 1

	designs_all = True if temp > 1 else False

	temp = 0
	for i in platform:
		temp += 1

	platform_all = True if temp > 1 else False

	temp = 0
	for i in sku:
		temp += 1

	sku_all = True if temp > 1 else False

	temp = 0
	for i in design_type:
		temp += 1

	design_type_all = True if temp > 1 else False


	# reports part
	display_block = 'none'

	if gen_rep_board != []:
		if show_report == 'yes':
			
			display_block = 'block'

			for i in range(len(platform)):
				if str(platform[i][0]) not in input_platform:
					platform[i][2] = ""

			for i in range(len(sku)):
				if str(sku[i][0]) not in input_sku:
					sku[i][2] = ""

			for i in range(len(design_type)):
				if str(design_type[i][0]) not in input_design_type:
					design_type[i][2] = ""

			for i in range(len(review_phase)):
				if str(review_phase[i][0]) not in input_review_phase:
					review_phase[i][2] = ""

			for i in range(len(designs)):
				if str(designs[i][0]) not in input_designs:
					designs[i][2] = ""

			platform.sort(key = lambda x: x[2],reverse=True)
			sku.sort(key = lambda x: x[2],reverse=True)
			design_type.sort(key = lambda x: x[2],reverse=True)
			review_phase.sort(key = lambda x: x[2],reverse=True)
			designs.sort(key = lambda x: x[2],reverse=True)
	else:
		gen_rep_board = [2,3,4]	# just hard coded with existing design to display none in html page

	color = []
	f = open("colours2.txt","r")
	s = f.readlines()
	for i in s:
		i2 = i.split("\n")
		color.append(i2[0].strip())	

	sql="SELECT * from DataMiningViewTable where BoardID IN %s ORDER BY BoardID DESC"
	val=(gen_rep_board,)
	signoffdetail=execute_query(sql,val)

	areaissue_arr = []
	ratio={}
	degree={}
	area=[]
	total_count=0
	sql="SELECT BR.BoardID, BR.AreaofIssue, COUNT(BR.ParentCommentID) from BoardDetails  BD , BoardReview BR  ,ScheduleTable ST , ScheduleStatusType SST where BD.BoardID IN %s AND BR.ParentCommentID = 0 AND BR.AreaOfIssue <> '' AND BD.BoardID = ST.BoardID  AND BD.BoardID =BR.BoardID AND ST.ScheduleStatusID  =SST.ScheduleID GROUP BY BR.AreaofIssue"
	val=(gen_rep_board,)
	areaissue=execute_query(sql,val)
	for i in range(len(areaissue)):
		total_count=total_count+areaissue[i][2]
		areaissue_arr.append(areaissue[i][2])

	cnt=0
	for i in range(len(areaissue)):
		ratio[areaissue[i][1]]=str(((areaissue[i][2]/total_count)*360))+"deg"
		cnt=cnt+areaissue[i][2]
		degree[areaissue[i][1]] = str(cnt*360/total_count)+"deg"
		a = str((areaissue[i][2] / total_count) * 100)
		area.append(a[0:4])


	total_count2=0
	comp_array=[]
	area2=[]

	sql="SELECT BR.BoardID, BR.ComponentID,CT.ComponentName, COUNT(BR.ParentCommentID) from ComponentType CT,BoardDetails  BD, BoardReview BR, ScheduleTable ST, ScheduleStatusType SST where BD.BoardID IN %s AND BR.AreaOfIssue <> '' AND BR.RiskLevel = 'High' AND BR.ComponentID=CT.ComponentID and BR.ParentCommentID = 0 AND BD.BoardID = ST.BoardID AND BD.BoardID = BR.BoardID AND ST.ScheduleStatusID = SST.ScheduleID GROUP BY BR.ComponentID"
	val=(gen_rep_board,)
	components=execute_query(sql,val)

	for i in range(len(components)):
		total_count2=total_count2+components[i][3]
		comp_array.append(components[i][3])

	cnt2 = 0
	for i in range(len(components)):
		ratio[components[i][1]] = str(((components[i][3] / total_count2) * 360)) + "deg"
		cnt2 = cnt2 + components[i][3]
		#degree.append(str(cnt2 * 360 / total_count2) + "deg")
		a = str((components[i][3] / total_count2)*100)
		area2.append(a[0:4])

	total_count3=0
	comp_array3=[]
	area3=[]

	sql="SELECT BR.BoardID, BR.ComponentID,CT.ComponentName, COUNT(BR.ParentCommentID) from ComponentType CT,BoardDetails  BD, BoardReview BR, ScheduleTable ST, ScheduleStatusType SST where BD.BoardID IN %s AND BR.AreaOfIssue <> '' AND BR.ComponentID=CT.ComponentID and BR.ParentCommentID = 0 AND BD.BoardID = ST.BoardID AND BD.BoardID = BR.BoardID AND ST.ScheduleStatusID = SST.ScheduleID GROUP BY BR.ComponentID"
	val=(gen_rep_board,)
	components3=execute_query(sql,val)

	for i in range(len(components3)):
		total_count3=total_count3+components3[i][3]
		comp_array3.append(components3[i][3])

	cnt3 = 0
	for i in range(len(components3)):
		ratio[components3[i][1]] = str(((components3[i][3] / total_count3) * 360)) + "deg"
		cnt3 = cnt3 + components3[i][3]
		#degree.append(str(cnt2 * 360 / total_count2) + "deg")
		a = str((components3[i][3] / total_count3)*100)
		area3.append(a[0:4])

	a1 = list(zip(areaissue,area))
	a2 = list(zip(components,area2))	
	a3 = list(zip(components3,area3))	


	a1 = sorted(a1, key = lambda x:(float(x[1])),reverse = True)	
	a2 = sorted(a2, key = lambda x:(float(x[1])),reverse = True)
	a3 = sorted(a3, key = lambda x:(float(x[1])),reverse = True)

	if(len(a1)>10):
			a1 = a1[0:10]
	if(len(a2)>10):
			a2 = a2[0:10]	
	if(len(a3)>10):
			a3 = a3[0:10]	

	areaissue_arr.sort(reverse = True)
	if(len(areaissue_arr)>10):
		areaissue_arr = areaissue_arr[0:10]

	comp_array.sort(reverse = True)
	if(len(comp_array)>10):
		comp_array = comp_array[0:10]

	comp_array3.sort(reverse = True)
	if(len(comp_array3)>10):
		comp_array3 = comp_array3[0:10]

	return render("data_mining_gen_reports.html",user_role_name=user_role_name,region_name=region_name,display_block=display_block,has_data=has_data,designs_all=designs_all,platform_all=platform_all,sku_all=sku_all,design_type_all=design_type_all,designs=designs,platform=platform,sku=sku,design_type=design_type,review_phase=review_phase,username=username,start_date_default=start_date_default,end_date_default=end_date_default,start_date_min=start_date_min,start_date_max=start_date_max,end_date_min=end_date_min,end_date_max=end_date_max,start_date_default_ww=start_date_default_ww,end_date_default_ww=end_date_default_ww,signoffdetail=signoffdetail, ratio=ratio,degree=degree, area2=area2,area3=area3,area=area,components=components,components3=components3, areaissue=areaissue, color=color,comp_array=comp_array,comp_array3=comp_array3, areaissue_arr=areaissue_arr,a1 = a1,a2 = a2,a3 = a3)

@app.route("/get_sku_name",methods = ['POST', 'GET'])
def get_sku_name():

	array_body = request.form.get("array_body")
	data = json.loads(array_body)
	start_calender_date = data[0]
	end_calender_date = data[1]
	platform = data[2]

	final_result = {}

	sql="SELECT DISTINCT d1.BoardID,b1.BoardName,b1.PlatformID,p1.PlatformName,b1.SKUID,s1.SKUName,b1.DesignTypeID,d2.DesignTypeName,b1.ReviewTimelineID,r1.ReviewTimelineName FROM DesignCalendar d1,BoardDetails b1,BoardReview b2,Platform p1,SUK s1,DesignType d2,ReviewTimeline r1 WHERE b1.BoardID = d1.BoardID AND b1.BoardID = b2.BoardID AND p1.PlatformID = b1.PlatformID AND b1.SKUID = s1.SKUID AND b1.DesignTypeID = d2.DesignTypeID AND b1.ReviewTimelineID = r1.ReviewTimelineID AND d1.ProposedEndDate >= %s AND d1.ProposedStartDate <= %s AND b2.DesignDocument <> '' AND b1.PlatformID IN %s order by d1.BoardID desc"
	val=(start_calender_date,end_calender_date,platform)
	result=execute_query(sql,val)

	designs = []
	platform = []
	sku = []
	design_type = []
	review_phase = []
	temp_board_id = []

	designs_temp = []
	platform_temp = []
	sku_temp = []
	design_type_temp = []
	review_phase_temp = []

	has_data = False

	if result != ():
		has_data = True

		for i in range(len(result)):
			designs_temp.append([result[i][0],result[i][1],"checked"])
			platform_temp.append([result[i][2],result[i][3],"checked"])
			sku_temp.append([result[i][4],result[i][5],"checked"])
			design_type_temp.append([result[i][6],result[i][7],"checked"])
			review_phase_temp.append([result[i][8],result[i][9],"checked"])

	platform_temp = list(frozenset(tuple(row) for row in platform_temp)) 
	sku_temp = list(frozenset(tuple(row) for row in sku_temp)) 
	design_type_temp = list(frozenset(tuple(row) for row in design_type_temp)) 
	review_phase_temp = list(frozenset(tuple(row) for row in review_phase_temp)) 	

	for row in platform_temp:
		platform.append(list(row))

	for row in sku_temp:
		sku.append(list(row))

	for row in design_type_temp:
		design_type.append(list(row))

	for row in review_phase_temp:
		review_phase.append(list(row))

	for row in designs_temp:
		designs.append(list(row))									

	# sorting
	platform.sort(key = lambda x: x[1])
	sku.sort(key = lambda x: x[1])
	design_type.sort(key = lambda x: x[1])
	review_phase.sort(key = lambda x: x[1])

	temp = 0
	for i in designs:
		temp += 1

	designs_all = True if temp > 1 else False

	temp = 0
	for i in platform:
		temp += 1

	platform_all = True if temp > 1 else False

	temp = 0
	for i in sku:
		temp += 1

	sku_all = True if temp > 1 else False

	temp = 0
	for i in design_type:
		temp += 1

	design_type_all = True if temp > 1 else False


	final_result['platform'] = platform
	final_result['sku'] = sku
	final_result['design_type'] = design_type
	final_result['review_phase'] = review_phase
	final_result['design_name'] = designs
	
	final_result['designs_all'] = designs_all
	final_result['platform_all'] = platform_all
	final_result['sku_all'] = sku_all
	final_result['design_type_all'] = design_type_all

	return jsonify(final_result)

@app.route("/get_design_type",methods = ['POST', 'GET'])
def get_design_type():

	array_body = request.form.get("array_body")
	data = json.loads(array_body)
	start_calender_date = data[0]
	end_calender_date = data[1]
	platform = data[2]
	sku = data[3]

	final_result = {}

	sql="SELECT DISTINCT d1.BoardID,b1.BoardName,b1.PlatformID,p1.PlatformName,b1.SKUID,s1.SKUName,b1.DesignTypeID,d2.DesignTypeName,b1.ReviewTimelineID,r1.ReviewTimelineName FROM DesignCalendar d1,BoardDetails b1,BoardReview b2,Platform p1,SUK s1,DesignType d2,ReviewTimeline r1 WHERE b1.BoardID = d1.BoardID AND b1.BoardID = b2.BoardID AND p1.PlatformID = b1.PlatformID AND b1.SKUID = s1.SKUID AND b1.DesignTypeID = d2.DesignTypeID AND b1.ReviewTimelineID = r1.ReviewTimelineID AND d1.ProposedEndDate >= %s AND d1.ProposedStartDate <= %s AND b2.DesignDocument <> '' AND b1.PlatformID IN %s AND b1.SKUID IN %s order by d1.BoardID desc"
	val=(start_calender_date,end_calender_date,platform,sku)
	result=execute_query(sql,val)

	designs = []
	platform = []
	sku = []
	design_type = []
	review_phase = []
	temp_board_id = []

	designs_temp = []
	platform_temp = []
	sku_temp = []
	design_type_temp = []
	review_phase_temp = []

	has_data = False

	if result != ():
		has_data = True

		for i in range(len(result)):
			designs_temp.append([result[i][0],result[i][1],"checked"])
			platform_temp.append([result[i][2],result[i][3],"checked"])
			sku_temp.append([result[i][4],result[i][5],"checked"])
			design_type_temp.append([result[i][6],result[i][7],"checked"])
			review_phase_temp.append([result[i][8],result[i][9],"checked"])

	platform_temp = list(frozenset(tuple(row) for row in platform_temp)) 
	sku_temp = list(frozenset(tuple(row) for row in sku_temp)) 
	design_type_temp = list(frozenset(tuple(row) for row in design_type_temp)) 
	review_phase_temp = list(frozenset(tuple(row) for row in review_phase_temp)) 	

	for row in platform_temp:
		platform.append(list(row))

	for row in sku_temp:
		sku.append(list(row))

	for row in design_type_temp:
		design_type.append(list(row))

	for row in review_phase_temp:
		review_phase.append(list(row))

	for row in designs_temp:
		designs.append(list(row))									

	# sorting
	platform.sort(key = lambda x: x[1])
	sku.sort(key = lambda x: x[1])
	design_type.sort(key = lambda x: x[1])
	review_phase.sort(key = lambda x: x[1])

	temp = 0
	for i in designs:
		temp += 1

	designs_all = True if temp > 1 else False

	temp = 0
	for i in platform:
		temp += 1

	platform_all = True if temp > 1 else False

	temp = 0
	for i in sku:
		temp += 1

	sku_all = True if temp > 1 else False

	temp = 0
	for i in design_type:
		temp += 1

	design_type_all = True if temp > 1 else False


	final_result['platform'] = platform
	final_result['sku'] = sku
	final_result['design_type'] = design_type
	final_result['review_phase'] = review_phase
	final_result['design_name'] = designs
	
	final_result['designs_all'] = designs_all
	final_result['platform_all'] = platform_all
	final_result['sku_all'] = sku_all
	final_result['design_type_all'] = design_type_all

	return jsonify(final_result)

@app.route("/get_review_phase",methods = ['POST', 'GET'])
def get_review_phase():

	array_body = request.form.get("array_body")
	data = json.loads(array_body)
	start_calender_date = data[0]
	end_calender_date = data[1]
	platform = data[2]
	sku = data[3]
	design_type = data[4]

	final_result = {}

	sql="SELECT DISTINCT d1.BoardID,b1.BoardName,b1.PlatformID,p1.PlatformName,b1.SKUID,s1.SKUName,b1.DesignTypeID,d2.DesignTypeName,b1.ReviewTimelineID,r1.ReviewTimelineName FROM DesignCalendar d1,BoardDetails b1,BoardReview b2,Platform p1,SUK s1,DesignType d2,ReviewTimeline r1 WHERE b1.BoardID = d1.BoardID AND b1.BoardID = b2.BoardID AND p1.PlatformID = b1.PlatformID AND b1.SKUID = s1.SKUID AND b1.DesignTypeID = d2.DesignTypeID AND b1.ReviewTimelineID = r1.ReviewTimelineID AND d1.ProposedEndDate >= %s AND d1.ProposedStartDate <= %s AND b2.DesignDocument <> '' AND b1.PlatformID IN %s AND b1.SKUID IN %s AND b1.DesignTypeID IN %s order by d1.BoardID desc"
	val=(start_calender_date,end_calender_date,platform,sku,design_type)
	result=execute_query(sql,val)

	designs = []
	platform = []
	sku = []
	design_type = []
	review_phase = []
	temp_board_id = []

	designs_temp = []
	platform_temp = []
	sku_temp = []
	design_type_temp = []
	review_phase_temp = []

	has_data = False

	if result != ():
		has_data = True

		for i in range(len(result)):
			designs_temp.append([result[i][0],result[i][1],"checked"])
			platform_temp.append([result[i][2],result[i][3],"checked"])
			sku_temp.append([result[i][4],result[i][5],"checked"])
			design_type_temp.append([result[i][6],result[i][7],"checked"])
			review_phase_temp.append([result[i][8],result[i][9],"checked"])

	#designs = list(frozenset(tuple(row) for row in designs))
	platform_temp = list(frozenset(tuple(row) for row in platform_temp)) 
	sku_temp = list(frozenset(tuple(row) for row in sku_temp)) 
	design_type_temp = list(frozenset(tuple(row) for row in design_type_temp)) 
	review_phase_temp = list(frozenset(tuple(row) for row in review_phase_temp)) 	

	for row in platform_temp:
		platform.append(list(row))

	for row in sku_temp:
		sku.append(list(row))

	for row in design_type_temp:
		design_type.append(list(row))

	for row in review_phase_temp:
		review_phase.append(list(row))

	for row in designs_temp:
		designs.append(list(row))									

	# sorting
	platform.sort(key = lambda x: x[1])
	sku.sort(key = lambda x: x[1])
	design_type.sort(key = lambda x: x[1])
	review_phase.sort(key = lambda x: x[1])

	temp = 0
	for i in designs:
		temp += 1

	designs_all = True if temp > 1 else False

	temp = 0
	for i in platform:
		temp += 1

	platform_all = True if temp > 1 else False

	temp = 0
	for i in sku:
		temp += 1

	sku_all = True if temp > 1 else False

	temp = 0
	for i in design_type:
		temp += 1

	design_type_all = True if temp > 1 else False


	final_result['platform'] = platform
	final_result['sku'] = sku
	final_result['design_type'] = design_type
	final_result['review_phase'] = review_phase
	final_result['design_name'] = designs
	
	final_result['designs_all'] = designs_all
	final_result['platform_all'] = platform_all
	final_result['sku_all'] = sku_all
	final_result['design_type_all'] = design_type_all

	return jsonify(final_result)

@app.route("/get_designs",methods = ['POST', 'GET'])
def get_designs():

	array_body = request.form.get("array_body")
	data = json.loads(array_body)
	start_calender_date = data[0]
	end_calender_date = data[1]
	platform = data[2]
	sku = data[3]
	design_type = data[4]
	review_phase = data[5]

	final_result = {}

	sql="SELECT DISTINCT d1.BoardID,b1.BoardName,b1.PlatformID,p1.PlatformName,b1.SKUID,s1.SKUName,b1.DesignTypeID,d2.DesignTypeName,b1.ReviewTimelineID,r1.ReviewTimelineName FROM DesignCalendar d1,BoardDetails b1,BoardReview b2,Platform p1,SUK s1,DesignType d2,ReviewTimeline r1 WHERE b1.BoardID = d1.BoardID AND b1.BoardID = b2.BoardID AND p1.PlatformID = b1.PlatformID AND b1.SKUID = s1.SKUID AND b1.DesignTypeID = d2.DesignTypeID AND b1.ReviewTimelineID = r1.ReviewTimelineID AND d1.ProposedEndDate >= %s AND d1.ProposedStartDate <= %s AND b2.DesignDocument <> '' AND b1.PlatformID IN %s AND b1.SKUID IN %s AND b1.DesignTypeID IN %s AND b1.ReviewTimelineID IN %s order by d1.BoardID desc"
	val=(start_calender_date,end_calender_date,platform,sku,design_type,review_phase)
	result=execute_query(sql,val)

	designs = []
	platform = []
	sku = []
	design_type = []
	review_phase = []
	temp_board_id = []

	designs_temp = []
	platform_temp = []
	sku_temp = []
	design_type_temp = []
	review_phase_temp = []

	has_data = False

	if result != ():
		has_data = True

		for i in range(len(result)):
			designs_temp.append([result[i][0],result[i][1],"checked"])
			platform_temp.append([result[i][2],result[i][3],"checked"])
			sku_temp.append([result[i][4],result[i][5],"checked"])
			design_type_temp.append([result[i][6],result[i][7],"checked"])
			review_phase_temp.append([result[i][8],result[i][9],"checked"])

	#designs = list(frozenset(tuple(row) for row in designs))
	platform_temp = list(frozenset(tuple(row) for row in platform_temp)) 
	sku_temp = list(frozenset(tuple(row) for row in sku_temp)) 
	design_type_temp = list(frozenset(tuple(row) for row in design_type_temp)) 
	review_phase_temp = list(frozenset(tuple(row) for row in review_phase_temp)) 	

	for row in platform_temp:
		platform.append(list(row))

	for row in sku_temp:
		sku.append(list(row))

	for row in design_type_temp:
		design_type.append(list(row))

	for row in review_phase_temp:
		review_phase.append(list(row))

	for row in designs_temp:
		designs.append(list(row))									

	# sorting
	platform.sort(key = lambda x: x[1])
	sku.sort(key = lambda x: x[1])
	design_type.sort(key = lambda x: x[1])
	review_phase.sort(key = lambda x: x[1])

	temp = 0
	for i in designs:
		temp += 1

	designs_all = True if temp > 1 else False

	temp = 0
	for i in platform:
		temp += 1

	platform_all = True if temp > 1 else False

	temp = 0
	for i in sku:
		temp += 1

	sku_all = True if temp > 1 else False

	temp = 0
	for i in design_type:
		temp += 1

	design_type_all = True if temp > 1 else False


	final_result['platform'] = platform
	final_result['sku'] = sku
	final_result['design_type'] = design_type
	final_result['review_phase'] = review_phase
	final_result['design_name'] = designs
	
	final_result['designs_all'] = designs_all
	final_result['platform_all'] = platform_all
	final_result['sku_all'] = sku_all
	final_result['design_type_all'] = design_type_all

	return jsonify(final_result)

@app.route('/request_history.html',methods = ['POST', 'GET'])
def request_history():

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'request_history'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	wwid=session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	query = "SELECT a.RequestID,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,c.Username,DesignManagerWWID,d.Username,e.Username,Comments,StartDate,EndDate,core,a.WWID,b.Username,IFNULL(f.BoardID,''),a.RefBoardID,a.RefBoardName FROM BoardDetailsRequest a NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) LEFT JOIN HomeTable b ON a.WWID = b.WWID LEFT JOIN HomeTable c ON a.DesignLeadWWID = c.WWID LEFT JOIN HomeTable d ON a.CADLeadWWID = d.WWID LEFT JOIN HomeTable e ON a.PIFLeadWWID = e.WWID LEFT JOIN RequestMap f ON a.RequestID=f.RequestID WHERE BoardStateID in (1,2) ORDER BY a.RequestID DESC"
	details_rs = execute_query_sql(query)

	details = []
	for row in details_rs:
		temp = []
		temp.append(row[0])
		temp.append(row[1])
		temp.append(row[2])
		temp.append(row[3])
		temp.append(row[4])
		temp.append(row[5])
		temp.append(row[6])
		temp.append(row[7])
		temp.append(row[8])
		temp.append(row[9])
		temp.append(row[10])
		temp.append(row[11])
		temp.append(row[12])
		temp.append(row[13])
		temp.append(row[14])
		temp.append(row[15])
		temp.append(row[16])
		temp.append(row[17])
		temp.append(row[18])

		# ww - start date
		temp.append(get_work_week_fun_with_year(date_value=row[13]))	#19
		temp.append(get_work_week_fun_with_year(date_value=row[14]))	#20

		if row[19] not in [0,"0"]:										#21 - reference design
			text_temp = "[ID: "+str(row[19])+"] - "+str(row[20])
			temp.append(text_temp)
		else:
			temp.append(str(row[20]))

		details.append(temp)

	sql="select RoleID from HomeTable where WWID=%s "
	val = (wwid,)
	role=execute_query(sql,val)[0][0]

	if (role == 14):
		mgt_access=True
	else:
		mgt_access=False

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	return render("request_history.html",is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,details=details,mgt_access=mgt_access,username=username,user_role_name=user_role_name,region_name=region_name,is_admin=is_admin)

@app.route("/check_design_combination",methods = ['POST', 'GET'])
def check_design_combination():

	data = request.form.get("data").split(',')

	core = data[4].replace(" ","+")

	sql = 'SELECT DesignTypeID FROM DesignType WHERE DesignTypeName = %s'
	val = (data[0],)
	designtype=execute_query(sql,val)

	designtypeID = 0
	if designtype != ():
		designtypeID = designtype[0][0]

	sql = 'SELECT PlatformID FROM Platform WHERE PlatformName = %s'
	val = (data[1],)
	platform=execute_query(sql,val)

	platformID = 0
	if platform != ():
		platformID = platform[0][0]

	sql = 'SELECT SKUID FROM SUK WHERE SKUName = %s'
	val = (data[2],)
	sku=execute_query(sql,val)

	skuID = 0
	if sku != ():
		skuID = sku[0][0]

	sql = 'SELECT MemTypeID FROM MemType WHERE MemTypeName = %s'
	val = (data[3],)
	memorytype=execute_query(sql,val)

	memorytypeID = 0
	if memorytype != ():
		memorytypeID = memorytype[0][0]


	sql="SELECT a.RequestID,b.BoardStateName,c.ReviewTimelineName FROM BoardDetailsRequest a LEFT JOIN BoardState b ON a.BoardStateID = b.BoardStateID LEFT JOIN ReviewTimeline c ON c.ReviewTimelineID = a.ReviewTimelineID WHERE DesignTypeID=%s AND PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND core = %s AND a.RequestID > (SELECT MAX(d.RequestID) - 3 FROM BoardDetailsRequest d) ORDER BY a.RequestID"
	val = (designtypeID,platformID,skuID,memorytypeID,core)
	rs=execute_query(sql,val)

	result = []
	if rs != ():
		for row in rs:
			result.append(row)

	return jsonify(result)	

@app.route("/get_work_week",methods = ['POST', 'GET'])
def get_work_week():
	date_value = datetime.datetime.strptime(request.form.get("date_val"), '%Y-%m-%d')
	result = []
	result.append(float(str(get_isocalendar(date_value)[1])+'.'+str(get_isocalendar(date_value)[2])))
	return jsonify(result)

@app.route("/get_work_week_with_year",methods = ['POST', 'GET'])
def get_work_week_with_year():
	date_value = datetime.datetime.strptime(request.form.get("date_val"), '%Y-%m-%d')
	result = []
	result.append('WW'+str(float(str(get_isocalendar(date_value)[1])+'.'+str(get_isocalendar(date_value)[2])))+"'"+str(get_isocalendar(date_value)[0])[2:4])
	return jsonify(result)

def get_work_week_fun_with_year(date_value):
	try:
		return str('WW'+str(float(str(get_isocalendar(date_value)[1])+'.'+str(get_isocalendar(date_value)[2])))+"'"+str(get_isocalendar(date_value)[0])[2:4])
	except:
		if is_logging:
			logging.exception('')
		print("date error..")
		return ' '

def get_work_week_fun(date_value):
	try:
		return str(float(str(get_isocalendar(date_value)[1])+'.'+str(get_isocalendar(date_value)[2])))
	except:
		if is_logging:
			logging.exception('')
		print("date error..")
		return ' '

def get_isocalendar(date):

	try:
		temp = date.isocalendar()

		year = copy.deepcopy(temp[0])
		work_week_number = copy.deepcopy(temp[1])
		weekday_number = copy.deepcopy(temp[2])

		if (work_week_number == 53) and (year == 2020):
			work_week_number = 1
			year += 1

		#elif year == 2021:
		elif year in [2021,2022]:
			work_week_number += 1

			if work_week_number == 53:
				work_week_number = 1
				year += 1

		result = (year,work_week_number,weekday_number)

	except:
		print('date error in WW calc')
		result = (1,1,1)

	return result

@app.route("/get_design_tape_out_date",methods = ['POST', 'GET'])
def get_design_tape_out_date():
	data = json.loads(request.form.get("data"))

	start_date = datetime.datetime.strptime(data[1], '%Y-%m-%d')

	if data[0] == "Yes":
		end_date = get_work_week_addition(date_value=start_date,no_of_days=3).date()
	else:
		end_date = get_work_week_addition(date_value=start_date,no_of_days=6).date()

	end_date = str(end_date)
	return jsonify(end_date)

def get_work_week_addition(date_value,no_of_days):

	for i in range(no_of_days):

		if i == 0:
			EndDate = date_value + datetime.timedelta(days=1)
		else:
			EndDate += datetime.timedelta(days=1)

		if str(get_isocalendar(EndDate)[2]) in ['6','7']:
			EndDate += datetime.timedelta(days=1)

		if str(get_isocalendar(EndDate)[2]) in ['6','7']:
			EndDate += datetime.timedelta(days=1)

	return EndDate

def get_work_week_date_fmt(date_value):
	try:
		return str('WW'+str(get_isocalendar(date_value)[1])+'.'+str(get_isocalendar(date_value)[2])+"'"+str(get_isocalendar(date_value)[0])[2:4])
	except:
		if is_logging:
			logging.exception('')
		print("date error..")
		return ' '


def get_work_week_str_fmt(date_value):
	try:
		date_fmt_value = datetime.datetime.strptime(date_value, '%Y-%m-%d')
		return str('WW'+str(get_isocalendar(date_fmt_value)[1])+'.'+str(get_isocalendar(date_fmt_value)[2])+"'"+str(get_isocalendar(date_fmt_value)[0])[2:4])
	except:
		if is_logging:
			logging.exception('')
		print("date error..")
		return ' '

def get_date_from_work_week_fun(year,ww):
	print("......................")
	print(Week(int(year), ww).monday())
	print(Week(int(year), ww).monday().year)
	if Week(int(year), ww).monday().year != year:
		year += 1

	print(".+++++++++++++++.....................")
	print(Week(int(year), ww).monday())
	print(Week(int(year), ww).monday().year)

	return Week(int(year), ww).monday()

@app.route("/get_design_dates",methods = ['POST', 'GET'])
def get_design_dates():
	design_name = request.form.get("design_name")

	sql="SELECT BD.BoardID, DC.ProposedStartDate, DC.ProposedEndDate,DC.BoardState from BoardDetails  BD , DesignCalendar DC where BD.BoardID = DC.BoardID and BD.BoardID = %s LIMIT 1"
	val=(design_name,)
	dates=execute_query(sql,val)

	result = []
	result.append(dates[0][1].strftime('%Y-%m-%d'))
	result.append(dates[0][2].strftime('%Y-%m-%d'))
	result.append(dates[0][3])

	return jsonify(result)

@app.route("/design_files",methods = ['POST', 'GET'])
def design_files(boardid=None,comp_selected_list=[],my_designs="My",my_interfaces="All"):
	wwid=session.get('wwid')
	#wwid = '10644414'
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	if wwid is None:
		print("invalid sso login")
		session['target_page'] = 'design_files'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]
	
	is_admin = False
	if has_admin_access == "yes":
		is_admin = True

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	if ((has_admin_access == "yes") or (is_design_owner) or (is_layout_owner)):
		pass
	else:
		return render('error_custom.html',error='You do not have access to this page. Please contact admin.',username=username,user_role_name=user_role_name,region_name=region_name)

	if boardid == None:
		if request.method == 'POST':
			boardid = request.form.get("boardid", type=int)
			comp_selected_list = request.form.getlist("comp_select")

			print("boardid: ",boardid)
			print(type(boardid))

			print("comp_selected_list: ",comp_selected_list)
			print(type(comp_selected_list))

			if comp_selected_list is None:
				comp_selected_list = []

			my_designs = request.form.get("my_designs")

			if my_designs is None:
				my_designs = "My"

	data = {}

	if boardid == None:
		boardid = 0

	if comp_selected_list == None:
		comp_selected_list = []


	sql = "SELECT AreaofIssue FROM AreaOfIssue ORDER BY AreaofIssue"
	area = execute_query_sql(sql)

	areas=[]
	for i in area:
		areas.append(i[0])

	design_status_list = []
	design_list = []
	my_designs_id = []


	temp_data = []
	temp_data = get_my_designs()
	my_designs_all_checked = ""
	my_designs_my_checked = "checked"

	for i in range(0,len(temp_data[1])):
		my_designs_id.append(temp_data[1][i][0])

	if my_designs == "All":
		temp_data = []
		temp_data = get_all_designs()
		my_designs_all_checked = "checked"
		my_designs_my_checked = ""


	#design_status_list = get_status_list_sorted(data_list=temp_data[0])
	design_status_list = get_order_status_list(list=temp_data[0])
	design_list = temp_data[1]

	comp_status_list = []
	comp_list = []

	if my_interfaces == "All":
		temp_data = []
		temp_data = get_all_interfaces(boardid=boardid)
		my_interfaces_all_checked = "checked"
		my_interfaces_my_checked = ""

	else:
		temp_data = []
		temp_data = get_my_interfaces(boardid=boardid)
		my_interfaces_all_checked = ""
		my_interfaces_my_checked = "checked"	

	#comp_status_list = get_status_list_sorted(data_list=temp_data[0])
	comp_status_list = get_order_status_list(list=temp_data[0])
	comp_list = temp_data[1]

	# for listing my designs with dates
	boards = []
	ww_date = []
	boards,ww_date = get_listed_status_designs(my_designs_id=my_designs_id)
	
	# to get Intrfaces level details
	for compid in comp_selected_list:
		data = get_design_data_page(data=data,boardid=boardid,compid=compid,complist=comp_list)

	# edit permission check
	file_upload_edit_enabled = False

	# design file details
	sql = "SELECT IFNULL(a.BoardFileName,''),IFNULL(a.SchematicsName,''),IFNULL(a.StackupFileName,''),IFNULL(a.LengthReportFileName,''),IFNULL(a.OthersFileName,''),IFNULL(b.Comments,''),IFNULL(c.UserName,''),IFNULL(a.Insert_Time,''),a.Count FROM UploadDesignFiles a LEFT JOIN UploadReSubmit b ON a.BoardID = b.BoardID LEFT JOIN HomeTable c on c.WWID = a.WWID WHERE a.BoardID = %s"
	val = (boardid,)
	file_details = execute_query(sql,val)

	design_document_name_list = []

	if file_details != ():
		file_details_available = True
		file_details_colspan = 4

		design_document_name_list.append([file_details[0][0],"Board File"])
		design_document_name_list.append([file_details[0][1],"Schematics File"])
		design_document_name_list.append([file_details[0][2],"Stackup File"])
		design_document_name_list.append([file_details[0][3],"Lenght Report File"])
		design_document_name_list.append([file_details[0][4],"Others File"])

	else:
		file_details_available = False
		file_details_colspan = 2+1

		# initializing if there are no records
		file_details = (("","","","",""))

	design_document_name_list = [row for row in design_document_name_list if row[0] != '']

	sql = "SELECT a.ScheduleStatusID FROM ScheduleTable a WHERE a.BoardID = %s AND a.ScheduleStatusID IN (2,3,6)"
	val = (boardid,)
	design_status_check = execute_query(sql,val)

	if design_status_check != ():

		if has_admin_access == "yes":
			file_upload_edit_enabled = True

		if ((is_design_owner) or (is_layout_owner)):
			# only my designs user should be editable
			if boardid in my_designs_id:
				file_upload_edit_enabled = True

	# to keep view mode for these 2 design
	if boardid in [60,61,'60','61']:
		file_upload_edit_enabled = False

	if not file_upload_edit_enabled:
		file_details_colspan -= 1

	# for left side display
	sql = "SELECT DISTINCT b.ComponentName FROM BoardReviewDesigner a, ComponentType b WHERE a.BoardID = %s AND a.ComponentID = b.ComponentID AND a.IsPdgDesignSubmitted = %s"
	val = (boardid,"yes")
	appl_comp_list = execute_query(sql,val)

	applicable_comp_name_list = []
	for i in range(0,len(appl_comp_list)):
		applicable_comp_name_list.append(appl_comp_list[i][0])

	board_name = ''
	is_rev0p6_design = False
	is_rev1p0_design = True

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID,BoardName,ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	if sku_plat != ():
		board_name = sku_plat[0][4]

		if sku_plat[0][5] in [1,'1']:
			is_rev0p6_design = True
			is_rev1p0_design = False

	if boardid == 0:
		design_list_table_show = "block"
		up_arrow_btn = "block"
		down_arrow_btn = "none"
	else:
		design_list_table_show = "none"
		up_arrow_btn = "none"
		down_arrow_btn = "block"

	return render("design_files.html",is_rev0p6_design=is_rev0p6_design,is_rev1p0_design=is_rev1p0_design,user_role_name=user_role_name,down_arrow_btn=down_arrow_btn,up_arrow_btn=up_arrow_btn,design_list_table_show=design_list_table_show,boards=boards,region_name=region_name,board_name=board_name,applicable_comp_name_list=applicable_comp_name_list,ww_date=ww_date,is_admin=is_admin,is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,data=data,boardid=boardid,my_designs_id=my_designs_id,design_document_name_list=design_document_name_list,my_designs_all_checked=my_designs_all_checked,my_designs_my_checked=my_designs_my_checked,my_interfaces_all_checked=my_interfaces_all_checked,my_interfaces_my_checked=my_interfaces_my_checked,file_upload_edit_enabled=file_upload_edit_enabled,file_details=file_details,file_details_colspan=file_details_colspan,file_details_available=file_details_available,areas=areas,comp_selected_list=comp_selected_list,username = username,design_status_list=design_status_list,design_list=design_list,comp_status_list=comp_status_list,comp_list=comp_list)

@app.route("/get_designs_ajax",methods = ['POST', 'GET'])
def get_designs_ajax():

	wwid=session.get('wwid')
	username = session.get('username')

	data = request.form.get("data")

	result = {}

	design_status_list = []
	design_list = []

	if data == "All":
		temp_data = []
		temp_data = get_all_designs()

	else:
		temp_data = []
		temp_data = get_my_designs()

	#result["design_status_list"] = json.dumps(get_status_list_sorted(data_list=temp_data[0]))
	result["design_status_list"] = json.dumps(get_order_status_list(list=temp_data[0]))
	result["design_list"] = json.dumps(temp_data[1])

	return jsonify(result)

@app.route("/get_import_designs",methods = ['POST', 'GET'])
def get_import_designs():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]
	comp_id = data["comp_id"]

	design_list = []
	design_status_list = []

	sql = "SELECT PlatformID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	platformid = execute_query(sql,val)

	if platformid != ():
		sql = "SELECT DISTINCT a.BoardID,a.BoardName,c.ScheduleTypeName FROM BoardDetails a, ScheduleTable b,ScheduleStatusType c,BoardReview d WHERE a.BoardID <> %s AND a.PlatformID = %s AND a.BoardID=b.BoardID AND b.ScheduleStatusID=c.ScheduleID AND a.BoardID = d.BoardID AND d.ComponentID = %s AND d.AreaOfIssue IS NOT NULL AND d.AreaOfIssue <> '' AND d.Submitted = 'yes' AND (d.HasChild = 'no' OR d.HasChild = '' OR d.HasChild IS NULL) AND b.ScheduleStatusID <> 3 ORDER BY FIELD(b.ScheduleStatusID,2,6,3,7,5,1,4),a.BoardID DESC"
		val = (boardid,platformid[0][0],comp_id)
		result = execute_query(sql,val)

		if result != ():
			for i in range(0,len(result)):
				temp = [result[i][0],result[i][1],result[i][2]]
				design_list.append(temp)
				design_status_list.append(result[i][2])

		design_status_list = set(design_status_list)
		design_status_list = list(design_status_list)

	return jsonify([design_status_list,design_list])


@app.route("/get_import_issue_status",methods = ['POST', 'GET'])
def get_import_issue_status():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	import_boardid = data["import_boardid"]
	comp_id = data["comp_id"]

	issue_list = []

	sql = "SELECT DISTINCT IFNULL(NULLIF(IssueStatus,''),'Open') FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND Submitted = %s AND (HasChild = 'no' OR HasChild = '' OR HasChild IS NULL)"
	val = (import_boardid,comp_id,'yes')
	result = execute_query(sql,val)

	if result != ():
		for i in range(0,len(result)):
			issue_list.append(result[i][0])

	return jsonify(issue_list)

def get_all_designs():

	wwid=session.get('wwid')
	username = session.get('username')

	design_list = []
	design_status_list = []

	sql = "SELECT DISTINCT c.ScheduleTypeName FROM BoardDetails a, ScheduleTable b,ScheduleStatusType c WHERE a.BoardID=b.BoardID AND b.ScheduleStatusID=c.ScheduleID ORDER BY FIELD(b.ScheduleStatusID,2,6,3,7,5,1,4),a.BoardID DESC"
	result = execute_query_sql(sql)

	if result != ():
		for i in range(0,len(result)):
			design_status_list.append(result[i][0])


	sql = "SELECT a.BoardID,a.BoardName,c.ScheduleTypeName FROM BoardDetails a, ScheduleTable b,ScheduleStatusType c WHERE a.BoardID=b.BoardID AND b.ScheduleStatusID=c.ScheduleID ORDER BY FIELD(b.ScheduleStatusID,2,6,3,7,5,1,4),a.BoardID DESC"
	result = execute_query_sql(sql)

	if result != ():
		for i in range(0,len(result)):
			temp = [result[i][0],result[i][1],result[i][2]]
			design_list.append(temp)

	design_status_list = get_order_status_list(list=design_status_list)

	return [design_status_list,design_list]

def get_my_designs():

	wwid=session.get('wwid')
	username = session.get('username')

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	is_admin = False
	if has_admin_access == "yes":
		is_admin = True

	design_list = []
	design_status_list = []

	sql = "SELECT B.BoardID FROM BoardDetails B,ScheduleStatusType S1, ScheduleTable S2 WHERE S2.BoardID = B.BoardID AND S1.ScheduleID = S2.ScheduleStatusID ORDER BY FIELD(S2.ScheduleStatusID,2,6,3,7,5,1,4),B.BoardID DESC"
	bnames = execute_query_sql(sql)

	for j in bnames:

		sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
		val = (j[0],)
		sku_plat = execute_query(sql,val)

		present = False
		if is_admin:
			present = True

		if(present == False):
			sql = "SELECT * FROM BoardDetails WHERE BoardID = %s AND ((DesignLeadWWID = %s) OR (CADLeadWWID = %s) OR (PIFLeadWWID = %s))"
			val = (j[0],wwid,wwid,wwid)
			result = execute_query(sql,val)

			if result != ():
				present = True	

		if(present == False):
			sql = "SELECT * FROM BoardDetails a LEFT JOIN HomeTable b ON a.DesignManagerWWID=b.UserName WHERE BoardID = %s AND b.WWID = %s"
			val = (j[0],wwid)
			result = execute_query(sql,val)

			if result != ():
				present = True	

		# for component review
		if(present == False):

			sql = "SELECT C3.CategoryLeadWWID,C2.PrimaryWWID,C2.SecondaryWWID,C3.CategoryLeadWWID1 FROM  ComponentReview C2,CategoryLeadTable C3,ComponentType C1 WHERE C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C1.ComponentID = C2.ComponentID AND C1.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID=%s"
			val = (sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
			primary_des = execute_query(sql,val)
			for i in primary_des:
				if(str(i[0]) == wwid or str(i[1]) == wwid or wwid in str(i[2]) or wwid in str(i[3])):
					present = True
					break

		# for component design
		if(present == False):

			sql = "SELECT C3.CategoryLeadWWID,C2.PrimaryWWID,C2.SecondaryWWID,C3.CategoryLeadWWID1 FROM  ComponentDesign C2,CategoryLeadTable C3,ComponentType C1 WHERE C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C1.ComponentID = C2.ComponentID AND C1.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID=%s"
			val = (sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
			primary_des = execute_query(sql,val)
			for i in primary_des:
				if(str(i[0]) == wwid or str(i[1]) == wwid or wwid in str(i[2]) or wwid in str(i[3])):
					present = True
					break

		if(present == True):
			sql = "SELECT B.BoardID,B.BoardName,S1.ScheduleTypeName,S1.ScheduleID FROM BoardDetails B, HomeTable H, DesignCalendar D2, ScheduleStatusType S1, ScheduleTable S2 WHERE B.DesignLeadWWID = H.WWID AND B.BoardID = D2.BoardID AND S2.BoardID = B.BoardID AND S1.ScheduleID = S2.ScheduleStatusID AND B.BoardID = %s ORDER BY S1.ScheduleTypeName  "
			val = (j[0],)
			blist = execute_query(sql,val)

			if blist != ():
				for i in range(0,len(blist)):
					design_list.append([blist[i][0],blist[i][1],blist[i][2]])
					
					if blist[i][2] not in design_status_list:
						#if blist[i][3] not in [3,'3']:
						design_status_list.append(blist[i][2])

	design_status_list = get_order_status_list(list=design_status_list)

	return [design_status_list,design_list]

@app.route("/get_interfaces_ajax",methods = ['POST', 'GET'])
def get_interfaces_ajax():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]
	btn_value = data["btn"]

	result = {}

	comp_status_list = []
	comp_list = []

	if btn_value == "All":
		temp_data = []
		temp_data = get_all_interfaces(boardid=boardid)

	else:
		temp_data = []
		temp_data = get_my_interfaces(boardid=boardid)

	result["comp_list"] = json.dumps(temp_data[1])
	#result["comp_status_list"] = json.dumps(get_status_list_sorted(data_list=temp_data[0]))
	result["comp_status_list"] = json.dumps(get_order_status_list(list=temp_data[0]))

	return jsonify(result)

def get_all_interfaces(boardid=0):

	wwid=session.get('wwid')
	username = session.get('username')

	comp_status_list = []
	comp_list = []

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	if sku_plat != ():
		sql = "SELECT DISTINCT S2.ScheduleTypeName FROM BoardReviewDesigner B1, ComponentType C1, ComponentReview C2, ScheduleTableComponent S1,ScheduleStatusType S2 WHERE B1.ComponentID = C1.ComponentID AND B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND C2.IsValid = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID = %s AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S2.ScheduleID = S1.ScheduleStatusID ORDER BY FIELD(S1.ScheduleStatusID,2,6,3,7,5,1,4), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
		val = (boardid,True,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		result = execute_query(sql,val)

		if result != ():
			for i in range(0,len(result)):
				comp_status_list.append(result[i][0])

		sql = "SELECT DISTINCT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1, ComponentType C1, ComponentReview C2, ScheduleTableComponent S1,ScheduleStatusType S2 WHERE B1.ComponentID = C1.ComponentID AND B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND C2.IsValid = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID = %s AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S2.ScheduleID = S1.ScheduleStatusID ORDER BY FIELD(S1.ScheduleStatusID,2,6,3,7,5,1,4), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
		val = (boardid,True,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		complist = execute_query(sql,val)

		if complist != ():
			for i in range(0,len(complist)):
				temp = [complist[i][0],complist[i][1],complist[i][2],complist[i][3]]
				comp_list.append(temp)

		sql = "SELECT CR.ComponentID,CT.ComponentName,CR.PrimaryWWID,CR.SecondaryWWID FROM  BoardDetails BD , ComponentReview CR,ComponentType CT WHERE BD.BoardID = %s AND BD.PlatformID=CR.PlatformID AND BD.SKUID=CR.SKUID AND BD.MemTypeID=CR.MemTypeID AND BD.DesignTypeID=CR.DesignTypeID AND CR.ComponentID=CT.ComponentID AND CR.IsValid = %s ORDER BY ComponentID"
		val = (boardid,True)
		comp_rew_dets = execute_query(sql, val)

		temp_all_list = []
		if comp_list != ():
			for i in range(0,len(comp_list)):
				temp_all_list.append(comp_list[i][0])

		for i in range(0,len(comp_rew_dets)):
			if comp_rew_dets[i][0] not in temp_all_list:
				#if (comp_rew_dets[i][2] != 99999999) or ((comp_rew_dets[i][3] != "['']") and (comp_rew_dets[i][3] != "[99999999]") and (comp_rew_dets[i][3] != "[]")):
				if comp_rew_dets[i][2] != 99999999:
					comp_status_list.append('Yet_to_Kickstart')
					comp_list.append([comp_rew_dets[i][0],comp_rew_dets[i][1],'Yet_to_Kickstart',3])

		comp_status_list = set(comp_status_list)
		comp_status_list = list(comp_status_list)

		comp_status_list = get_order_status_list(list=comp_status_list)

	return [comp_status_list,comp_list]

def get_my_interfaces(boardid=0):

	wwid=session.get('wwid')
	username = session.get('username')

	comp_list = []
	comp_status_list = []

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	if sku_plat != ():
		sql = "SELECT DISTINCT S2.ScheduleTypeName FROM BoardReviewDesigner B1, ComponentType C1, ComponentReview C2, ScheduleTableComponent S1,ScheduleStatusType S2 WHERE B1.ComponentID = C1.ComponentID AND B1.BoardID = %s AND C2.IsValid = %s AND %s IN (SELECT DISTINCT PrimaryWWID FROM ComponentReview C2 WHERE B1.ComponentID = C2.ComponentID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID =%s AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S2.ScheduleID = S1.ScheduleStatusID  UNION SELECT DISTINCT CT1.CategoryLeadWWID FROM CategoryLeadTable CT1,ComponentReview C2,ComponentType C5 WHERE C2.ComponentID = B1.ComponentID AND C2.ComponentID = C5.ComponentID AND C5.CategoryID = CT1.CategoryID AND CT1.SKUID = %s AND CT1.PlatformID = %s AND CT1.MemTypeID =%s AND CT1.DesignTypeID = %s) ORDER BY FIELD(S1.ScheduleStatusID,2,6,3,7,5,1,4), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
		val = (boardid,True,wwid,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		result = execute_query(sql,val)

		if result != ():
			for i in range(0,len(result)):
				comp_status_list.append(result[i][0])

		sql = "SELECT DISTINCT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1, ComponentType C1, ComponentReview C2, ScheduleTableComponent S1,ScheduleStatusType S2 WHERE B1.ComponentID = C1.ComponentID AND B1.BoardID = %s AND C2.IsValid = %s AND %s IN (SELECT DISTINCT PrimaryWWID FROM ComponentReview C2 WHERE B1.ComponentID = C2.ComponentID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID =%s AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S2.ScheduleID = S1.ScheduleStatusID  UNION SELECT DISTINCT CT1.CategoryLeadWWID FROM CategoryLeadTable CT1,ComponentReview C2,ComponentType C5 WHERE C2.ComponentID = B1.ComponentID AND C2.ComponentID = C5.ComponentID AND C5.CategoryID = CT1.CategoryID AND CT1.SKUID = %s AND CT1.PlatformID = %s AND CT1.MemTypeID =%s AND CT1.DesignTypeID = %s) ORDER BY FIELD(S1.ScheduleStatusID,2,6,3,7,5,1,4), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
		val = (boardid,True,wwid,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		complist = execute_query(sql,val)

		if complist != ():
			for i in range(0,len(complist)):
				temp = [complist[i][0],complist[i][1],complist[i][2],complist[i][3]]
				comp_list.append(temp)

	comp_status_list = get_order_status_list(list=comp_status_list)

	return [comp_status_list,comp_list]

def get_status_list_sorted(data_list=[]):

	temp_data_list = []

	if data_list != []:

		for row in data_list:
			if row == "Yet_to_Kickstart":
				temp_data_list.append(row)

			if row == "Ongoing":
				temp_data_list.append(row)

			if row == "Reopened":
				temp_data_list.append(row)

			if row == "No_Signoff":
				temp_data_list.append(row)

			if row == "Rejected":
				temp_data_list.append(row)

			if row == "Signoff_Overdue":
				temp_data_list.append(row)

			if row == "Signed-Off":
				temp_data_list.append(row)

	return temp_data_list

def get_design_data_page(data,boardid,compid,complist):

	is_admin = session.get('is_admin')
	wwid=session.get('wwid')
	username = session.get('username')

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	data[compid] = {}
	data[compid]["add_new_issues_btn_enable"] = False
	feedbacks = []
	saved_feedbacks = []
	data[compid]["comp_status_id"] = 3

	if complist != []:
		for i in range(0,len(complist)):
			if complist[i][0] == int(compid):
				data[compid]["comp_name"] = complist[i][1]
				data[compid]["comp_status_id"] = complist[i][3]
				data[compid]["comp_status"] = complist[i][2]

				if data[compid]["comp_status_id"] in [3]:
					sql = "SELECT ScheduleStatusID FROM ScheduleTable WHERE BoardID = %s"
					val = (boardid,)
					board_status = execute_query(sql,val)[0][0]

					if board_status in [2,3]:
						data[compid]["add_new_issues_btn_enable"] = True

	# to keep view mode for these 2 design
	if boardid in [60,61,'60','61']:
		data[compid]["add_new_feedbacks_btn_enable"] = False

	# Graded by Design Team As PDG data
	sql = "SELECT DISTINCT IFNULL(B1.PDG,''),IFNULL(B1.CommentDesigner,'') FROM BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.ComponentID = %s" 
	val = (boardid,compid)
	result = execute_query(sql,val)

	data[compid]["comp_pdg"] = ''
	data[compid]["pdg_comment"] = ''

	if result != ():
		data[compid]["comp_pdg"] = result[0][0]
		data[compid]["pdg_comment"] = result[0][1]

	data[compid]["pdg_not_met_checked"] = ""
	data[compid]["pdg_met_checked"] = ""

	if data[compid]["comp_pdg"] == "Met":
		data[compid]["pdg_met_checked"] = "checked"
		data[compid]["pdg_not_met_checked"] = ""

	elif data[compid]["comp_pdg"] == "Not Met":
		data[compid]["pdg_not_met_checked"] = "checked"
		data[compid]["pdg_met_checked"] = ""

	# list of edit discard feedbacks at board level
	sql = "SELECT BoardID,ComponentID,CommentID,WWIDreviewer,WWIDdesigner FROM BoardReviewTemp WHERE BoardID = %s"
	val = (boardid,)
	edit_discard_rs = execute_query(sql,val)

	# feedbacks data
	sql = "SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,B1.Comment_Reviewer,B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time,B2.PDG,B2.CommentDesigner,'',S.ScheduleStatusID,B1.IsImported,B1.ImportedFromDesignID FROM BoardReview B1 INNER JOIN BoardReviewDesigner B2 ON B1.BoardID = B2.BoardID AND B1.ComponentID = B2.ComponentID INNER JOIN ScheduleTable S ON B1.BoardID = S.BoardID INNER JOIN DesignCalendar D5 ON B1.BoardID = D5.BoardID WHERE B1.ParentCommentID = 0 AND B1.Submitted = 'yes' AND B1.AddedByDesignTeam = 'yes' AND B1.BoardID = %s AND B1.ComponentID = %s ORDER BY B1.CommentID" 
	val = (boardid,compid)
	parents = execute_query(sql,val)

	#sql = "SELECT DISTINCT B1.*,B2.PDG,B2.CommentDesigner,U.ReviewFilenames,S.ScheduleStatusID FROM BoardReview B1 INNER JOIN BoardReviewDesigner B2 ON B1.BoardID = B2.BoardID AND B1.ComponentID = B2.ComponentID INNER JOIN UploadTable U ON B1.BoardID = U.BoardID AND B1.ComponentID = U.ComponentID INNER JOIN ScheduleTable S ON B1.BoardID = S.BoardID INNER JOIN DesignCalendar D5 ON B1.BoardID = D5.BoardID WHERE B1.ParentCommentID <> 0 AND B1.Submitted = 'yes' AND B1.BoardID = %s AND B1.ComponentID = %s ORDER BY BoardID,ComponentID,ParentCommentID,Submit_Time,CommentID,Submitted"
	#sql = "SELECT DISTINCT B1.*,B2.PDG,B2.CommentDesigner,U.ReviewFilenames,S.ScheduleStatusID FROM BoardReview B1 INNER JOIN BoardReviewDesigner B2 ON B1.BoardID = B2.BoardID AND B1.ComponentID = B2.ComponentID INNER JOIN UploadSignOffFiles U ON B1.BoardID = U.BoardID AND B1.ComponentID = U.ComponentID INNER JOIN ScheduleTable S ON B1.BoardID = S.BoardID INNER JOIN DesignCalendar D5 ON B1.BoardID = D5.BoardID WHERE B1.ParentCommentID <> 0 AND B1.Submitted = 'yes' AND B1.BoardID = %s AND B1.ComponentID = %s ORDER BY BoardID,ComponentID,ParentCommentID,Submit_Time,CommentID,Submitted"
	#val = (boardid,compid)
	#children = execute_query(sql,val)

	result = []

	for i in parents:
		result.append(i)

	f_no = 0


	for i in range(0,len(result)):

		f_row_span = 1
		f_no_td_enabled = True

		if result[i][12] == 0:
			f_no += 1							

		temp = []
		temp.append(result[i][0])
		temp.append(result[i][1])
		temp.append(result[i][2])

		temp.append(f_no_td_enabled)
		temp.append(f_row_span)
		temp.append(f_no)

		temp.append(result[i][24])
		temp.append(result[i][3])
		temp.append(result[i][4].replace("\n","<br>"))
		temp.append(result[i][5])

		if result[i][6] != None:
			temp1 = result[i][6].replace("---------","<br>")
			temp2 = temp1.replace("--",'<br><br><span style="color:grey; font-size: smaller;">Updated By: ')
			temp3 = temp2.replace(", Date: ",'<br>Updated On: ')
			if (temp3.find('Updated By:') != -1):
				temp3 += '</span>'
			temp.append(temp3.replace("\n","<br>"))
		else:
			temp.append('')

		temp.append(result[i][7])
		temp.append(result[i][23].replace("\n","<br>"))

		#download attachement
		if result[i][26] not in ('No File',None,''):
			temp.append(True)
		else:
			temp.append(False)

		if result[i][8] != None:
			temp.append(result[i][8])
		else:
			temp.append('')

		if result[i][9] != None:
			temp1 = result[i][9].replace("---------","<br>")
			temp2 = temp1.replace("--",'<br><span style="color:grey; font-size: smaller;">Updated By: ')
			temp3 = temp2.replace(", Date: ",'<br>Updated On: ')
			if (temp3.find('Updated By:') != -1):
				temp3 += '</span>'
			temp.append(temp3.replace("\n","<br>"))
		else:
			temp.append('')				

		if result[i][27] not in ('No File',None,''):
			temp.append(True)
		else:
			temp.append(False)

		if result[i][19] != None:
			temp.append(result[i][19])
		else:
			temp.append('')

		if result[i][20] != None:
			temp1 = result[i][20].replace("---------","<br>")
			temp2 = temp1.replace("--",'<br><span style="color:grey; font-size: smaller;">Updated By: ')
			temp3 = temp2.replace(", Date: ",'<br>Updated On: ')
			if (temp3.find('Updated By:') != -1):
				temp3 += '</span>'
			temp.append(temp3.replace("\n","<br>"))
		else:
			temp.append('')

		if result[i][21] != None:
			temp.append(result[i][21])
		else:
			temp.append('')

		temp.append('') #bgcolor	#20

		temp.append(result[i][36]) # is imported

		temp.append(result[i][37]) # imported from design ID

		if result[i][36] == "yes":	# for non editable imported feedbacks
			temp.append('readonly')
		else:
			temp.append('')

		temp.append(result[i][16])	# subitted designer
		temp.append(result[i][18])	# subitted reviewer2
		# actual parent id
		if result[i][28] is None:
			temp.append("")
		else:
			temp.append(result[i][28])

		if result[i][26] not in ('No File',None,''):	#27
			temp.append("Download File\nFile Name: "+result[i][26])
		else:
			temp.append("")

		if result[i][27] not in ('No File',None,''):	#28
			temp.append("Download File\nFile Name: "+result[i][26])
		else:
			temp.append("")

		# 29, 30- edit discard flag
		temp.append("block")
		temp.append("none")

		feedbacks.append(temp)

	for i in range(0,len(feedbacks)):

		f_row_span = 0
		feedbacks[i][3] = True

		for j in range(i,len(feedbacks)):
			if str(feedbacks[i][5]) == str(feedbacks[j][5]):
				f_row_span += 1

		feedbacks[i][4] = copy.deepcopy(f_row_span)

		if i>0:
			if str(feedbacks[i][5]) == str(feedbacks[i-1][5]):
				feedbacks[i][3] = False
				feedbacks[i][20] = '#effbff'
				feedbacks[i-1][20] = '#effbff'

	data[compid]["feedbacks"] = copy.deepcopy(feedbacks)

	# for update flag of edit discard icon in UI
	for i in range(0,len(data[compid]["feedbacks"])):
		for j in range(0,len(edit_discard_rs)):
			if (edit_discard_rs[j][1] == data[compid]["feedbacks"][i][1]) and (edit_discard_rs[j][2] == data[compid]["feedbacks"][i][2]):
				if is_admin or is_elec_owner:
					if str(wwid) == str(edit_discard_rs[j][3]):
						data[compid]["feedbacks"][i][29] = "none"
						data[compid]["feedbacks"][i][30] = "block"

				if is_design_owner or is_layout_owner:
					if str(wwid) == str(edit_discard_rs[j][4]):
						data[compid]["feedbacks"][i][29] = "none"
						data[compid]["feedbacks"][i][30] = "block"

	# saved feedbacks
	sql = "SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,B1.Comment_Reviewer,B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time,B2.PDG,B2.CommentDesigner,'',S.ScheduleStatusID,B1.IsImported,B1.ImportedFromDesignID,B1.EditDiscardFlag FROM BoardReview B1 INNER JOIN BoardReviewDesigner B2 ON B1.BoardID = B2.BoardID AND B1.ComponentID = B2.ComponentID INNER JOIN ScheduleTable S ON B1.BoardID = S.BoardID INNER JOIN DesignCalendar D5 ON B1.BoardID = D5.BoardID WHERE B1.ParentCommentID = 0 AND B1.Saved = 'yes' AND B1.Submitted = 'no' AND B1.AddedByDesignTeam = 'yes' AND B1.BoardID = %s AND B1.ComponentID = %s ORDER BY B1.CommentID" 
	val = (boardid,compid)
	parents = execute_query(sql,val)

	result = []

	for i in parents:
		result.append(i)

	f_no = 0


	for i in range(0,len(result)):

		f_row_span = 1
		f_no_td_enabled = True

		if result[i][12] == 0:
			f_no += 1							

		temp = []
		temp.append(result[i][0])
		temp.append(result[i][1])
		temp.append(result[i][2])

		temp.append(f_no_td_enabled)
		temp.append(f_row_span)
		temp.append(f_no)

		temp.append(result[i][24])
		temp.append(result[i][3])
		temp.append(result[i][4])
		temp.append(result[i][5])

		temp.append(result[i][6])

		temp.append(result[i][7])
		temp.append(result[i][23])

		#download attachement
		if result[i][26] not in ('No File',None,''):
			temp.append(result[i][26])
		else:
			temp.append("No File")

		temp.append(result[i][8])

		temp.append(result[i][9])

		if result[i][27] not in ('No File',None,''):
			temp.append(result[i][27])
		else:
			temp.append("No File")

		temp.append(result[i][19])

		temp.append(result[i][20])

		temp.append(result[i][21])

		temp.append('') #bgcolor	#20

		temp.append(result[i][36]) # is imported

		temp.append(result[i][37]) # imported from design ID

		if result[i][36] == "yes":	# for non editable imported feedbacks
			temp.append('readonly')
		else:
			temp.append('')

		temp.append(result[i][16])	# subitted designer
		temp.append(result[i][18])	# subitted reviewer2
		# actual parent id
		if result[i][28] is None:
			temp.append("")
		else:
			temp.append(result[i][28])

		temp.append(result[i][38]) # edit discard flag

		saved_feedbacks.append(temp)

	for i in range(0,len(saved_feedbacks)):

		f_row_span = 0
		saved_feedbacks[i][3] = True

		for j in range(i,len(saved_feedbacks)):
			if str(saved_feedbacks[i][5]) == str(saved_feedbacks[j][5]):
				f_row_span += 1

		saved_feedbacks[i][4] = copy.deepcopy(f_row_span)

		if i>0:
			if str(saved_feedbacks[i][5]) == str(saved_feedbacks[i-1][5]):
				saved_feedbacks[i][3] = False
				saved_feedbacks[i][20] = '#effbff'
				saved_feedbacks[i-1][20] = '#effbff'

	# convert python to javascript json format
	data[compid]["saved_feedbacks"] = json.dumps(copy.deepcopy(saved_feedbacks))

	return data

@app.route("/get_design_import_issues",methods = ['POST', 'GET'])
def get_design_import_issues():
	
	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))

	board_id = copy.deepcopy(data['board_id'])
	comp_id = copy.deepcopy(data['comp_id'])
	import_boardid = copy.deepcopy(data['import_boardid'])
	import_issue_status = copy.deepcopy(data['import_issue_status'])
	imp_design_team_feedbacks = copy.deepcopy(data['imp_design_team_feedbacks'])

	if 'Open' in import_issue_status:
		sql = "SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,B1.Comment_Reviewer,B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time FROM BoardReview B1 WHERE B1.BoardID = %s AND B1.ComponentID = %s AND B1.Submitted = %s AND (((B1.AreaOfIssue <> '' AND B1.AreaOfIssue IS NOT NULL) AND ((B1.IssueStatus <> %s AND B1.IssueStatus <> %s) OR (B1.IssueStatus IS NULL))) OR (B1.IssueStatus IN %s)) ORDER BY B1.CommentID,B1.Submitted"
		val = (import_boardid,comp_id,'yes','Close','Conditional Waiver',import_issue_status)
		parents = execute_query(sql,val)

	else:
		sql = "SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,B1.Comment_Reviewer,B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time FROM BoardReview B1 WHERE B1.BoardID = %s AND B1.ComponentID = %s AND B1.Submitted = %s AND B1.IssueStatus IN %s ORDER BY B1.CommentID,B1.Submitted"
		val = (import_boardid,comp_id,'yes',import_issue_status)
		parents = execute_query(sql,val)

	result = []
	import_feedbacks = []

	for i in parents:
		result.append(i)

	f_no = 0


	for i in range(0,len(result)):

		f_row_span = 1
		f_no_td_enabled = True

		if result[i][12] == 0:
			f_no += 1							

		temp = []
		temp.append(result[i][0])
		temp.append(result[i][1])
		temp.append(result[i][2])
		#temp.append(0)

		temp.append(f_no_td_enabled)
		temp.append(f_row_span)
		temp.append(f_no)

		temp.append(result[i][24])
		temp.append(result[i][3])

		if result[i][4] is not None:
			temp.append(result[i][4].replace("&"," and "))
		else:
			temp.append('')

		temp.append(result[i][5])

		if result[i][6] is not None:
			temp.append(result[i][6].replace("&"," and "))
		else:
			temp.append('')

		temp.append(result[i][7])

		if result[i][23] is not None:
			temp.append(result[i][23].replace("&"," and "))
		else:
			temp.append('')

		#download attachement
		if result[i][26] not in ('No File',None,''):
			temp.append(result[i][26])
		else:
			temp.append("No File")

		# if Import Design team feedbacks selected yes
		if imp_design_team_feedbacks:
			temp.append(result[i][8])
			#temp.append(result[i][9])

			if result[i][9] is not None:
				temp_design_comments_a = result[i][9].replace("---------","")

				temp_design_comments = temp_design_comments_a.replace("&"," and ")

				if temp_design_comments.find("--") != -1:
					temp1_design_comments = temp_design_comments.split("--",1)
					temp.append(temp1_design_comments[0])
				else:
					temp.append(temp_design_comments)
			else:
				temp.append('')

			#temp.append(result[i][27])
			if result[i][27] not in ('No File',None,''):
				temp.append(result[i][26])
			else:
				temp.append("No File")

		else:
			temp.append('')
			temp.append('')
			temp.append("No File")

		temp.append(result[i][19])

		temp.append(result[i][20])

		temp.append(result[i][21])

		temp.append('') #bgcolor

		temp.append(result[i][24]) # import issues filename (Design Document Name)

		import_feedbacks.append(temp)

	sql = "SELECT BoardFileName,SchematicsName,StackupFileName,LengthReportFileName,OthersFileName FROM UploadDesignFiles WHERE BoardID = %s"
	val = (board_id,)
	filename_rs = execute_query(sql,val)

	sql = "SELECT BoardFileName,SchematicsName,StackupFileName,LengthReportFileName,OthersFileName FROM UploadDesignFiles WHERE BoardID = %s"
	val = (import_boardid,)
	import_filename_rs = execute_query(sql,val)

	for i in range(0,len(import_feedbacks)):

		f_row_span = 0
		import_feedbacks[i][3] = True

		for j in range(i,len(import_feedbacks)):
			if str(import_feedbacks[i][5]) == str(import_feedbacks[j][5]):
				f_row_span += 1

		import_feedbacks[i][4] = copy.deepcopy(f_row_span)

		if i>0:
			if str(import_feedbacks[i][5]) == str(import_feedbacks[i-1][5]):
				import_feedbacks[i][3] = False
				import_feedbacks[i][20] = '#effbff'
				import_feedbacks[i-1][20] = '#effbff'


		# to update design document name
		if filename_rs != ():
			if import_filename_rs != ():
				if import_feedbacks[i][21] == import_filename_rs[0][0]:
					import_feedbacks[i][21] = filename_rs[0][0]

				elif import_feedbacks[i][21] == import_filename_rs[0][1]:
					import_feedbacks[i][21] = filename_rs[0][1]

				elif import_feedbacks[i][21] == import_filename_rs[0][2]:
					import_feedbacks[i][21] = filename_rs[0][2]

				elif import_feedbacks[i][21] == import_filename_rs[0][3]:
					import_feedbacks[i][21] = filename_rs[0][3]

				elif import_feedbacks[i][21] == import_filename_rs[0][4]:
					import_feedbacks[i][21] = filename_rs[0][4]

				else:
					import_feedbacks[i][21] = ""
					#import_feedbacks[i][7] = ""
		else:
			import_feedbacks[i][21] = ""
			#import_feedbacks[i][7] = ""

	# convert python to javascript json format
	data["import_feedbacks"] = copy.deepcopy(import_feedbacks)

	sql = "SELECT AreaofIssue FROM AreaOfIssue ORDER BY AreaofIssue"
	area = execute_query_sql(sql)

	areas=[]
	for i in area:
		areas.append(i[0])

	data["areas"] = areas

	print(data)

	return jsonify(data)


@app.route("/save_design_files_data",methods = ['POST', 'GET'])
def save_design_files_data():

	wwid = session.get('wwid')
	username = session.get('username')

	form_data = json.loads(request.form.get("data"))

	data = {}
	data["comment_id_list"] = []
	for row in form_data:

		data[row["name"]] = row["value"]

	board_id = copy.deepcopy(data['board_id'])
	comp_id = copy.deepcopy(data['save_compid'])
	comp_selected_list = copy.deepcopy(data['comp_select[]'])
	pdg = copy.deepcopy(data["pdg_"+str(comp_id)])
	pdg_comments = copy.deepcopy(data["pdg_comments_"+str(comp_id)])
	new_issues_count = int(copy.deepcopy(data["new_issues_count_"+str(comp_id)]))
	issues_count = int(copy.deepcopy(data["issues_count_"+str(comp_id)]))

	current_time = datetime.datetime.now(tz)
	current_date =  datetime.datetime.now(tz).strftime('%Y-%m-%d')

	# log table
	try:
		log_notes = 'User has saved Designer PDG details for <br>Design ID: '+str(board_id)
		log_notes += '<br>Component Name: '+str(comp_id)+'<br>PDG: '+str(pdg)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Feedbacks',board_id,0,comp_id,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	sql = "SELECT EXISTS(SELECT * FROM BoardReviewDesigner WHERE ComponentID = %s and BoardID = %s)"
	val = (comp_id,board_id)
	isthere = execute_query(sql, val)

	if (isthere[0][0]):

		sql = "UPDATE BoardReviewDesigner SET PDG = %s,CommentDesigner = %s,CommentDesignUpdatedBy = %s WHERE ComponentID = %s AND BoardID = %s"
		val = (pdg,pdg_comments,wwid,comp_id, board_id)
		execute_query(sql, val)

	else:

		sql="INSERT INTO BoardReviewDesigner VALUES(%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
		val=(board_id,0,comp_id,pdg,pdg_comments,None,wwid,"",99999999,"",99999999,None,None)
		execute_query(sql,val)

		sql = "INSERT INTO ScheduleTableComponent VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE ScheduleStatusID = ScheduleStatusID"
		val = (board_id, comp_id, 3)
		execute_query(sql, val)

	#issues_count += 1

	for j in range(0,issues_count):

		try:
			comment_id = copy.deepcopy(data["comment_id_"+str(comp_id)+"_"+str(j)])
			import_comment_id = copy.deepcopy(data["import_comment_id_"+str(comp_id)+"_"+str(j)])
			is_imported_form = copy.deepcopy(data["is_imported_"+str(comp_id)+"_"+str(j)])
			imported_from = copy.deepcopy(data["imported_from_"+str(comp_id)+"_"+str(j)])
			design_doc_name = copy.deepcopy(data["design_doc_name_"+str(comp_id)+"_"+str(j)])
			design_doc_type = copy.deepcopy(data["design_doc_type_"+str(comp_id)+"_"+str(j)])
			signal_name = copy.deepcopy(data["signal_name_"+str(comp_id)+"_"+str(j)])
			area_of_issue = copy.deepcopy(data["area_of_issue_"+str(comp_id)+"_"+str(j)])
			feedback_summary = copy.deepcopy(data["feedback_summary_"+str(comp_id)+"_"+str(j)])
			feedback_ref = copy.deepcopy(data["feedback_ref_"+str(comp_id)+"_"+str(j)])
			impl_status = copy.deepcopy(data["impl_status_"+str(comp_id)+"_"+str(j)])
			design_comments = copy.deepcopy(data["design_comments_"+str(comp_id)+"_"+str(j)])

			if design_doc_type is not None:
				is_valid = True
			else:
				is_valid = False

		except Exception as inst:
			is_valid = False
			print(inst)

		if is_valid:
			
			if comment_id is None:
				comment_id = 0

			if import_comment_id is None:
				import_comment_id = 0

			comment_id = int(comment_id)
			import_comment_id = int(import_comment_id)
			DesignerFeedbackGiven = None
			Saved_Designer = None
			Submitted_Designer = None

			ReviewerFileName = 'No File'
			DesignerFileName = 'No File'
			
			is_imported = None
			imported_from_design_id = None
			imported_by = None

			if is_imported_form == "yes":
				is_imported = 'yes'
				imported_from_design_id = int(imported_from)
				imported_by = int(wwid)

				# if import is yes, then import_comment_id is ero, then which means imported data are saved and later user has loading the data, 
				# so in this case files are already uploaded, so we can map to current comment id
				if import_comment_id == 0:
					import_comment_id = copy.deepcopy(comment_id)

				sql = "SELECT ReviewerFileName,DesignerFileName FROM BoardReview WHERE CommentID = %s"
				val = (import_comment_id,)
				rs_file_names = execute_query(sql, val)

				if rs_file_names != ():
					ReviewerFileName = rs_file_names[0][0]
					DesignerFileName = rs_file_names[0][1]

			if ((len(design_comments) == 0) or (impl_status is None)):
				DesignerFeedbackGiven = None
				Saved_Designer = None
				Submitted_Designer = None
			else:			
				DesignerFeedbackGiven = "yes"
				Saved_Designer = "yes"
				Submitted_Designer = "no"
			

			BoardFileName = design_doc_name

			if comment_id == 0:
				max_feedback_no = 0

				sql = "INSERT INTO BoardReview(BoardID,ComponentID,DesignDocument,SignalName,AreaOfIssue,FeedbackSummary,RiskLevel,ImplementationStatus,Comment,ReviewerFeedbackGiven,DesignerFeedbackGiven,ParentCommentID,Saved,Submitted,Saved_Designer,Submitted_Designer,ReferenceNumber,BoardFileName,ReviewerFileName,DesignerFileName,ActualParentID,WWIDreviewer,WWIDdesigner,Submit_Time,AddedByDesignTeam,IsImported,ImportedFromDesignID,ImportedBy,UpdatedOnForDesignSection,FeedbackNo,is_edit_save_flag,is_edit_save_flag_design) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
				val = (board_id,comp_id,design_doc_type,signal_name,area_of_issue,feedback_summary,"",impl_status,design_comments,"yes",DesignerFeedbackGiven,0,"yes","no",Saved_Designer,Submitted_Designer,feedback_ref,BoardFileName,ReviewerFileName,DesignerFileName,0,wwid,wwid,current_time,'yes',is_imported,imported_from_design_id,imported_by,current_time,max_feedback_no,0,0)
				execute_query(sql,val)

				sql = "SELECT LAST_INSERT_ID()"
				comid =  execute_query_sql(sql)[0][0]

				temp = ["comment_id_"+str(comp_id)+"_"+str(j),comid]

				data["comment_id_list"].append(temp)

				# for import alone
				if is_imported_form == "yes":

					if import_comment_id != 0:

						sql = "INSERT INTO FileStorage (CommentID,ReviewerFilename,ReviewerFile,DesignerFilename,DesignerFile,Reviewer2Filename,Reviewer2File) SELECT %s,b.ReviewerFilename,b.ReviewerFile,b.DesignerFilename,b.DesignerFile,b.Reviewer2Filename,b.Reviewer2File FROM FileStorage b WHERE b.CommentID = %s"
						val = (comid,import_comment_id)
						execute_query(sql,val)

			else:
				# update table
				sql = "UPDATE BoardReview SET DesignDocument = %s,SignalName = %s,AreaOfIssue = %s,FeedbackSummary = %s,RiskLevel = %s,ImplementationStatus = %s,Comment = %s,ReviewerFeedbackGiven = %s,DesignerFeedbackGiven = %s,ParentCommentID = %s,Saved = %s,Submitted = %s,Saved_Designer = %s,Submitted_Designer = %s,ReferenceNumber = %s,BoardFileName = %s,ReviewerFileName = %s,DesignerFileName = %s,ActualParentID = %s,WWIDreviewer = %s,WWIDdesigner = %s,Submit_Time = %s,AddedByDesignTeam = %s,IsImported = %s,ImportedFromDesignID = %s,ImportedBy = %s,UpdatedOnForDesignSection = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
				val = (design_doc_type,signal_name,area_of_issue,feedback_summary,"",impl_status,design_comments,"yes",DesignerFeedbackGiven,0,"yes","no",Saved_Designer,Submitted_Designer,feedback_ref,BoardFileName,ReviewerFileName,DesignerFileName,0,wwid,wwid,current_time,'yes',is_imported,imported_from_design_id,imported_by,current_time,board_id,comp_id,comment_id)
				execute_query(sql,val)

	# to update for edit feedbacks
	sql = "SELECT CommentID,ParentCommentID FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND Submitted = %s"
	val = (board_id,comp_id,"yes")
	rs_edit_comment_id = execute_query(sql, val)

	if rs_edit_comment_id != ():
		for i in range(0,len(rs_edit_comment_id)):

			try:
				edit_design_doc_name = copy.deepcopy(data["edit_design_doc_name_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])])
				edit_design_doc_type = copy.deepcopy(data["edit_design_doc_type_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])])
				edit_signal_name = copy.deepcopy(data["edit_signal_name_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])])
				edit_area_of_issue = copy.deepcopy(data["edit_area_of_issue_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])])
				edit_feedback_summary = copy.deepcopy(data["edit_feedback_summary_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])])
				#edit_risk_level = request.form.get("edit_risk_level_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
				edit_feedback_ref = copy.deepcopy(data["edit_feedback_ref_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])])

				edit_impl_status = copy.deepcopy(data["edit_impl_status_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])])
				edit_comment = copy.deepcopy(data["edit_comment_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])])

				if ((edit_design_doc_name is not None) or (edit_impl_status is not None)):
					# save part
					# to backup submitted feedback data before saving edit data, to keep it for discard option for editing submitted data
					sql = "SELECT * FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (board_id,comp_id,rs_edit_comment_id[i][0])
					rs_temp = execute_query(sql,val)

					if rs_temp == ():	# for first time save only we are backing up submitted data
						sql = "INSERT INTO BoardReviewTemp SELECT * FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (board_id,comp_id,rs_edit_comment_id[i][0])
						execute_query(sql,val)

						sql = "UPDATE BoardReview SET EditDiscardFlag = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (1,board_id,comp_id,rs_edit_comment_id[i][0])
						execute_query(sql,val)

				# 1 st part
				if edit_design_doc_name is not None:
					print("updating 1")

					sql = "UPDATE BoardReview SET BoardFileName = %s, FeedbackSummary = %s, ReferenceNumber = %s, Saved = %s, Submitted = %s, ReviewerFeedbackGiven = %s,WWIDreviewer = %s, Submit_Time = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (edit_design_doc_name,edit_feedback_summary,edit_feedback_ref,"yes","no","no",wwid,t,board_id,comp_id,rs_edit_comment_id[i][0])
					execute_query(sql,val)

					# for child feedback check
					if edit_design_doc_type is not None:

						sql = "UPDATE BoardReview SET DesignDocument = %s, SignalName = %s, AreaOfIssue = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (edit_design_doc_type,edit_signal_name,edit_area_of_issue,board_id,comp_id,rs_edit_comment_id[i][0])
						execute_query(sql,val)		

					# to update edit file attachement for electrical 1sr part
					try:
						electrical_file = request.files["edit_electrical_file_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])]
						ReviewerFileName = electrical_file.filename

						if electrical_file is not None:
							if electrical_file.filename != '':
								file=electrical_file.read()
								filename=electrical_file.filename
								fname = board_name+"_"+comp_name+"_"+str(comp_id)+"_"+filename

								sql = "INSERT INTO FileStorage (CommentID,ReviewerFilename,ReviewerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE ReviewerFilename = %s, ReviewerFile = %s"
								val = (rs_edit_comment_id[i][0],fname,file,fname,file)
								execute_query(sql,val)

								sql = "UPDATE BoardReview SET ReviewerFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
								val = (filename,board_id,comp_id,rs_edit_comment_id[i][0])
								execute_query(sql,val)					

					except Exception as inst:
						print(inst)

				# 2nd part
				if edit_impl_status is not None:
					print("update 2")

					sql = "UPDATE BoardReview SET ImplementationStatus = %s, Comment = %s, DesignerFeedbackGiven = %s, Saved_Designer = %s, Submitted_Designer = %s, WWIDdesigner = %s, Submit_Time = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (edit_impl_status,edit_comment,"no","yes","no",wwid,t,board_id,comp_id,rs_edit_comment_id[i][0])
					execute_query(sql,val)

					# to update edit file attachement for electrical 1sr part
					try:
						designer_file = request.files["edit_designer_file_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])]
						DesignerFileName = designer_file.filename

						if designer_file is not None:
							if designer_file.filename != '':
								file=designer_file.read()
								filename=designer_file.filename
								fname = board_name+"_"+comp_name+"_"+str(comp_id)+"_"+filename

								sql = "INSERT INTO FileStorage (CommentID,DesignerFilename,DesignerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE DesignerFilename = %s, DesignerFile = %s"
								val = (rs_edit_comment_id[i][0],fname,file,fname,file)
								execute_query(sql,val)

								sql = "UPDATE BoardReview SET DesignerFilename = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
								val = (filename,board_id,comp_id,rs_edit_comment_id[i][0])
								execute_query(sql,val)		


					except Exception as inst:
						print(inst)

			except Exception as inst:
				print(inst)
				pass

	return jsonify(data)

@app.route("/design_files_submit_for_review",methods = ['POST', 'GET'])
def design_files_submit_for_review():

	wwid = session.get('wwid')
	username = session.get('username')

	board_id = request.form.get("board_id")
	comp_selected_list = eval(request.form.get("comp_select[]"))

	current_time = datetime.datetime.now(tz)
	current_date =  datetime.datetime.now(tz).strftime('%Y-%m-%d')
	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	# log table
	try:
		log_notes = 'User has updated Designer PDG details for <br>Design ID: '+str(board_id)
		log_wwid = session.get('wwid')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Feedbacks',board_id,0,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	query = "SELECT BoardName FROM BoardDetails WHERE BoardID=%s"
	val=(board_id,)
	board_name=execute_query(query,val)[0][0]

	new_comps = ''
	new_comps_id_list = []

	for comp_id in comp_selected_list:

		pdg = ''

		if request.form.get("pdg_"+str(comp_id)):
			pdg = request.form.get("pdg_"+str(comp_id))
			pdg_comments = request.form.get("pdg_comments_"+str(comp_id))
		
			new_issues_count = int(request.form.get("new_issues_count_"+str(comp_id)))
			issues_count = int(request.form.get("issues_count_"+str(comp_id)))

			query = "SELECT ComponentName FROM ComponentType WHERE ComponentID=%s"
			val=(comp_id,)
			comp_name=execute_query(query,val)[0][0]

			sql = "SELECT EXISTS(SELECT * FROM BoardReviewDesigner WHERE ComponentID = %s and BoardID = %s)"
			val = (comp_id,board_id)
			isthere = execute_query(sql, val)

			if (isthere[0][0]):

				# for saved new interface, then later submitted --> new interface added should be there in email trigger.
				sql = "SELECT * FROM BoardReviewDesigner WHERE ComponentID = %s and BoardID = %s AND IsPdgDesignSubmitted IS NULL"
				val = (comp_id,board_id)
				isthere_pdg = execute_query(sql, val)

				if isthere_pdg != ():
					new_comps += '<br>'+comp_name
					new_comps_id_list.append(comp_id)

				sql = "UPDATE BoardReviewDesigner SET PDG = %s,CommentDesigner = %s,CommentDesignUpdatedBy = %s,IsPdgDesignSubmitted = %s WHERE ComponentID = %s AND BoardID = %s"
				val = (pdg,pdg_comments,wwid,"yes",comp_id, board_id)
				execute_query(sql, val)

			else:

				new_comps += '<br>'+comp_name
				new_comps_id_list.append(comp_id)

				sql="INSERT INTO BoardReviewDesigner VALUES(%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
				val=(board_id,0,comp_id,pdg,pdg_comments,None,wwid,"",99999999,"",99999999,None,"yes")
				execute_query(sql,val)

				sql = "INSERT INTO ScheduleTableComponent VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE ScheduleStatusID = ScheduleStatusID"
				val = (board_id, comp_id, 3)
				execute_query(sql, val)


			# basic design quality check
			sql = "SELECT * FROM BasicDesignQualityCheck WHERE BoardID = %s AND ComponentID = %s AND QualityCheck = %s"
			val = (board_id,comp_id,"met")
			rs_check = execute_query(sql,val)

			if rs_check == ():

				sql = "SELECT CommentID,IsSubmitted FROM BasicDesignQualityCheck WHERE BoardID = %s AND ComponentID = %s ORDER BY CommentID DESC"
				val = (board_id,comp_id)
				rs_temp = execute_query(sql,val)

				is_quality_check_update_flag = True

				if rs_temp != ():

					for qlty_row in rs_temp:

						if qlty_row[1] != "yes":

							is_quality_check_update_flag = False

							sql = "UPDATE BasicDesignQualityCheck SET QualityCheck = %s,AreaOfQualityIssue = %s,Comments = %s,QualityNotMetCounter = %s,IsSubmitted = %s,UpdatedBy = %s,UpdatedOn = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
							val = ("",0,"",0,"",wwid,t,board_id,comp_id,qlty_row[0])
							execute_query(sql,val)

				if is_quality_check_update_flag:

					sql = "INSERT INTO BasicDesignQualityCheck(BoardID,ComponentID,QualityCheck,AreaOfQualityIssue,Comments,QualityNotMetCounter,IsSubmitted,UpdatedBy,UpdatedOn) VALUES(%s,%s,%s,%s,%s,%s,%s,%s,%s)"
					val = (board_id,comp_id,"",0,"",0,"",wwid,t)
					execute_query(sql,val)

			for j in range(0,issues_count):

				try:
					comment_id = request.form.get("comment_id_"+str(comp_id)+"_"+str(j))
					import_comment_id = request.form.get("import_comment_id_"+str(comp_id)+"_"+str(j))
					is_imported_form = request.form.get("is_imported_"+str(comp_id)+"_"+str(j))
					imported_from = request.form.get("imported_from_"+str(comp_id)+"_"+str(j))
					design_doc_name = request.form.get("design_doc_name_"+str(comp_id)+"_"+str(j))
					design_doc_type = request.form.get("design_doc_type_"+str(comp_id)+"_"+str(j))
					signal_name = request.form.get("signal_name_"+str(comp_id)+"_"+str(j))
					area_of_issue = request.form.get("area_of_issue_"+str(comp_id)+"_"+str(j))
					feedback_summary = request.form.get("feedback_summary_"+str(comp_id)+"_"+str(j))
					feedback_ref = request.form.get("feedback_ref_"+str(comp_id)+"_"+str(j))
					impl_status = request.form.get("impl_status_"+str(comp_id)+"_"+str(j))
					design_comments = request.form.get("design_comments_"+str(comp_id)+"_"+str(j))

					if design_doc_type is not None:
						is_valid = True
					else:
						is_valid = False

				except Exception as inst:
					is_valid = False
					print(inst)

				if is_valid:

					if comment_id is None:
						comment_id = 0		

					if import_comment_id is None:
						import_comment_id = 0

					comment_id = int(comment_id)
					import_comment_id = int(import_comment_id)
					DesignerFeedbackGiven = None
					Saved_Designer = None
					Submitted_Designer = None

					BoardFileName = copy.deepcopy(design_doc_name)

					try:
						electrical_file = request.files["electrical_file_"+str(comp_id)+"_"+str(j)]
					except:
						electrical_file = None

					try:	
						designer_file = request.files["designer_file_"+str(comp_id)+"_"+str(j)]
					except:
						designer_file = None

					ReviewerFileName = ''
					DesignerFileName = ''

					if electrical_file is not None:
						ReviewerFileName = copy.deepcopy(electrical_file.filename)

					if designer_file is not None:
						DesignerFileName = copy.deepcopy(designer_file.filename)
					
					is_imported = None
					imported_from_design_id = None
					imported_by = None

					if is_imported_form == "yes":
						is_imported = 'yes'
						imported_from_design_id = int(imported_from)
						imported_by = int(wwid)

						# after save, after sometime, page reloads, import_comment_id would be 0, and also import filenames would be already saved, so we are trying to fetch the same value here again
						if import_comment_id == 0:
							temp_import_comment_id = copy.deepcopy(comment_id)
						else:
							temp_import_comment_id = copy.deepcopy(import_comment_id)

						sql = "SELECT ReviewerFileName,DesignerFileName FROM BoardReview WHERE CommentID = %s"
						val = (temp_import_comment_id,)
						rs_file_names = execute_query(sql, val)

						if rs_file_names != ():
							ReviewerFileName = rs_file_names[0][0]
							DesignerFileName = rs_file_names[0][1]
					
					if ((len(design_comments) == 0) or (impl_status is None)):
						DesignerFeedbackGiven = None
						Saved_Designer = None
						Submitted_Designer = None
					else:			
						DesignerFeedbackGiven = "yes"
						Saved_Designer = "yes"
						Submitted_Designer = "yes"
					

					if ReviewerFileName == '':
						ReviewerFileName = 'No File'

					if DesignerFileName == '':
						DesignerFileName = 'No File'

					if is_imported == "yes":

						try:
							feedback_summary = feedback_summary.replace("---------","<br>")
							imported_from_design_id = request.form.get("imported_from_"+str(comp_id)+"_"+str(j))
							temp1 = feedback_summary.split("--",1)
							temp2 = temp1[1].split(", Date: ",1)
							temp_feedback_content = copy.deepcopy(temp1[0])	# it has feedback summary content
							temp_updated_by = copy.deepcopy(temp2[0])	# it has original username who has provided feedback

							sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
							val = (imported_from_design_id,)
							imported_from_design_name = execute_query(sql, val)[0][0]

							feedback_summary = temp_feedback_content+'<br><br>'+'<span style="color:grey; font-size: smaller;">Imported from: '+imported_from_design_name+' [ID:'+str(imported_from_design_id)+']'
							feedback_summary += '<br>Imported by: '+str(username)
							feedback_summary += '<br>Imported on: '+str(current_date)
							feedback_summary += '<br>Comments by: '+str(temp_updated_by)+'</span>'
						except:
							pass
					else:
						if feedback_summary is not None:
							feedback_summary += "--"+str(username)+", Date: "+str(current_date)

					if len(design_comments) != 0:

						if is_imported in [1,'1']:
							try:
								design_comments = design_comments.replace("---------","<br>")
								imported_from_design_id = request.form.get("imported_from_"+str(comp_id)+"_"+str(j))
								temp1 = design_comments.split("--",1)
								temp2 = temp1[1].split(", Date: ",1)
								temp_feedback_content = copy.deepcopy(temp1[0])	# it has feedback summary content
								temp_updated_by = copy.deepcopy(temp2[0])	# it has original username who has provided feedback

								design_comments = temp_feedback_content+'<br><br>'+'<span style="color:grey; font-size: smaller;">Imported from: '+imported_from_design_name+' [ID:'+str(imported_from_design_id)+']'
								design_comments += '<br>Imported by: '+str(username)
								design_comments += '<br>Imported on: '+str(current_date)
								design_comments += '<br>Comments by: '+str(temp_updated_by)+'</span>'
							except:
								pass
						else:
							design_comments += "--"+str(username)+", Date: "+str(current_date)

					# to get maximum feeback number for board and Interface
					sql = "SELECT IFNULL(MAX(FeedbackNo),0) FROM BoardReview WHERE BoardID = %s AND ComponentID = %s"
					val = (board_id,comp_id)
					max_feedback_no_rs = execute_query(sql, val)
					max_feedback_no = 1
					if max_feedback_no_rs != ():
						max_feedback_no = max_feedback_no_rs[0][0] + 1
					
					if comment_id == 0:

						sql = "INSERT INTO BoardReview(BoardID,ComponentID,DesignDocument,SignalName,AreaOfIssue,FeedbackSummary,RiskLevel,ImplementationStatus,Comment,ReviewerFeedbackGiven,DesignerFeedbackGiven,ParentCommentID,Saved,Submitted,Saved_Designer,Submitted_Designer,ReferenceNumber,BoardFileName,ReviewerFileName,DesignerFileName,ActualParentID,WWIDreviewer,WWIDdesigner,Submit_Time,AddedByDesignTeam,IsImported,ImportedFromDesignID,ImportedBy,UpdatedOnForDesignSection,FeedbackNo,is_edit_save_flag,is_edit_save_flag_design) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
						val = (board_id,comp_id,design_doc_type,signal_name,area_of_issue,feedback_summary,"",impl_status,design_comments,"yes",DesignerFeedbackGiven,0,"yes","yes",Saved_Designer,Submitted_Designer,feedback_ref,BoardFileName,ReviewerFileName,DesignerFileName,0,wwid,wwid,current_time,'yes',is_imported,imported_from_design_id,imported_by,current_time,max_feedback_no,0,0)
						execute_query(sql,val)

						sql = "SELECT LAST_INSERT_ID()"
						comment_id = copy.deepcopy(int(execute_query_sql(sql)[0][0]))

						# for import alone
						if is_imported_form == "yes":

							if import_comment_id != 0:

								sql = "INSERT INTO FileStorage (CommentID,ReviewerFilename,ReviewerFile,DesignerFilename,DesignerFile,Reviewer2Filename,Reviewer2File) SELECT %s,b.ReviewerFilename,b.ReviewerFile,b.DesignerFilename,b.DesignerFile,b.Reviewer2Filename,b.Reviewer2File FROM FileStorage b WHERE b.CommentID = %s"
								val = (comment_id,import_comment_id)
								execute_query(sql,val)

					else:
						# to get maximum feeback number for board and Interface
						sql = "SELECT IFNULL(FeedbackNo,0) FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (board_id,comp_id,comment_id)
						temp_feedbackno = execute_query(sql, val)

						update_feedback_no = 0

						if temp_feedbackno != ():
							if int(temp_feedbackno[0][0]) == 0:
								update_feedback_no = copy.deepcopy(max_feedback_no)
							else:
								update_feedback_no = copy.deepcopy(temp_feedbackno[0][0])

						# update table
						sql = "UPDATE BoardReview SET FeedbackNo = %s,DesignDocument = %s,SignalName = %s,AreaOfIssue = %s,FeedbackSummary = %s,RiskLevel = %s,ImplementationStatus = %s,Comment = %s,ReviewerFeedbackGiven = %s,DesignerFeedbackGiven = %s,ParentCommentID = %s,Saved = %s,Submitted = %s,Saved_Designer = %s,Submitted_Designer = %s,ReferenceNumber = %s,BoardFileName = %s,ReviewerFileName = %s,DesignerFileName = %s,ActualParentID = %s,WWIDreviewer = %s,WWIDdesigner = %s,Submit_Time = %s,AddedByDesignTeam = %s,IsImported = %s,ImportedFromDesignID = %s,ImportedBy = %s,UpdatedOnForDesignSection = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (update_feedback_no,design_doc_type,signal_name,area_of_issue,feedback_summary,"",impl_status,design_comments,"yes",DesignerFeedbackGiven,0,"yes","yes",Saved_Designer,Submitted_Designer,feedback_ref,BoardFileName,ReviewerFileName,DesignerFileName,0,wwid,wwid,current_time,'yes',is_imported,imported_from_design_id,imported_by,current_time,board_id,comp_id,comment_id)
						execute_query(sql,val)

						# to clear edit discard flag
						sql = "SELECT * FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (board_id,comp_id,comment_id)
						rs_temp = execute_query(sql,val)

						if rs_temp != ():
							sql = "UPDATE BoardReview SET EditDiscardFlag = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
							val = (0,board_id,comp_id,comment_id)
							execute_query(sql,val)

							sql = "DELETE FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
							val = (board_id,comp_id,comment_id)
							execute_query(sql,val)


					if is_imported_form != "yes":
						if electrical_file is not None:
							file=electrical_file.read()
							filename=electrical_file.filename
							fname = board_name+"_"+comp_name+"_"+str(j)+"_"+filename

							sql = "INSERT INTO FileStorage (CommentID,ReviewerFilename,ReviewerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE ReviewerFilename = %s, ReviewerFile = %s"
							val = (comment_id,fname,file,fname,file)
							execute_query(sql,val)

						if designer_file is not None:
							file=designer_file.read()
							filename=designer_file.filename
							fname = board_name+"_"+comp_name+"_"+str(j)+"_"+filename

							sql = "INSERT INTO FileStorage (CommentID,DesignerFilename,DesignerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE DesignerFilename = %s, DesignerFile = %s"
							val = (comment_id,fname,file,fname,file)
							execute_query(sql,val)

		is_edited_electrical = False
		is_edited_designer = False

		is_submit = True
		is_submitted = "yes"

		# to update for edit feedbacks
		sql = "SELECT CommentID,ParentCommentID FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND Submitted = %s"
		val = (board_id,comp_id,"yes")
		rs_edit_comment_id = execute_query(sql, val)


		if rs_edit_comment_id != ():
			for i in range(0,len(rs_edit_comment_id)):

				try:
					edit_design_doc_name = request.form.get("edit_design_doc_name_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
					edit_design_doc_type = request.form.get("edit_design_doc_type_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
					edit_signal_name = request.form.get("edit_signal_name_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
					edit_area_of_issue = request.form.get("edit_area_of_issue_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
					edit_feedback_summary = request.form.get("edit_feedback_summary_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
					#edit_risk_level = request.form.get("edit_risk_level_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
					edit_feedback_ref = request.form.get("edit_feedback_ref_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))

					edit_impl_status = request.form.get("edit_impl_status_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
					edit_comment = request.form.get("edit_comment_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))

					if ((edit_design_doc_name is not None) or (edit_impl_status is not None)):
						sql = "SELECT * FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (board_id,comp_id,rs_edit_comment_id[i][0])
						rs_temp = execute_query(sql,val)

						if rs_temp != ():
							sql = "UPDATE BoardReview SET EditDiscardFlag = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
							val = (0,board_id,comp_id,rs_edit_comment_id[i][0])
							execute_query(sql,val)

							sql = "DELETE FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
							val = (board_id,comp_id,rs_edit_comment_id[i][0])
							execute_query(sql,val)

					# 1 st part
					if edit_design_doc_name is not None:

						if is_submit:
							if edit_feedback_summary is not None:
								edit_feedback_summary += "--"+str(username)+", Date: "+str(current_date)

						sql = "UPDATE BoardReview SET BoardFileName = %s, FeedbackSummary = %s, ReferenceNumber = %s, Saved = %s, Submitted = %s, ReviewerFeedbackGiven = %s,WWIDreviewer = %s, Submit_Time = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (edit_design_doc_name,edit_feedback_summary,edit_feedback_ref,"yes",is_submitted,is_submitted,wwid,t,board_id,comp_id,rs_edit_comment_id[i][0])
						execute_query(sql,val)

						is_edited_electrical = True

						# for child feedback check
						if edit_design_doc_type is not None:

							sql = "UPDATE BoardReview SET DesignDocument = %s, SignalName = %s, AreaOfIssue = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
							val = (edit_design_doc_type,edit_signal_name,edit_area_of_issue,board_id,comp_id,rs_edit_comment_id[i][0])
							execute_query(sql,val)		


						# to update edit file attachement for electrical 1sr part
						try:
							electrical_file = request.files["edit_electrical_file_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])]
							ReviewerFileName = electrical_file.filename

							if electrical_file is not None:
								if electrical_file.filename != '':
									file=electrical_file.read()
									filename=electrical_file.filename
									fname = board_name+"_"+comp_name+"_"+str(comp_id)+"_"+filename

									sql = "INSERT INTO FileStorage (CommentID,ReviewerFilename,ReviewerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE ReviewerFilename = %s, ReviewerFile = %s"
									val = (rs_edit_comment_id[i][0],fname,file,fname,file)
									execute_query(sql,val)

									sql = "UPDATE BoardReview SET ReviewerFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
									val = (filename,board_id,comp_id,rs_edit_comment_id[i][0])
									execute_query(sql,val)					

						except Exception as inst:
							print(inst)

						if is_submit:
							is_electric_submitted = True

					# 2nd part
					if edit_impl_status is not None:
						
						if is_submit:
							edit_comment += "--"+str(username)+", Date: "+str(current_date)

							is_design_submitted = True

						sql = "UPDATE BoardReview SET ImplementationStatus = %s, Comment = %s, DesignerFeedbackGiven = %s, Saved_Designer = %s, Submitted_Designer = %s, WWIDdesigner = %s, Submit_Time = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (edit_impl_status,edit_comment,is_submitted,"yes",is_submitted,wwid,t,board_id,comp_id,rs_edit_comment_id[i][0])
						execute_query(sql,val)

						is_edited_designer = True

						# to update edit file attachement for electrical 1sr part
						try:
							designer_file = request.files["edit_designer_file_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])]
							DesignerFileName = designer_file.filename

							if designer_file is not None:
								if designer_file.filename != '':
									file=designer_file.read()
									filename=designer_file.filename
									fname = board_name+"_"+comp_name+"_"+str(comp_id)+"_"+filename

									sql = "INSERT INTO FileStorage (CommentID,DesignerFilename,DesignerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE DesignerFilename = %s, DesignerFile = %s"
									val = (rs_edit_comment_id[i][0],fname,file,fname,file)
									execute_query(sql,val)

									sql = "UPDATE BoardReview SET DesignerFilename = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
									val = (filename,board_id,comp_id,rs_edit_comment_id[i][0])
									execute_query(sql,val)		


						except Exception as inst:
							print(inst)

				except Exception as inst:
					print(inst)
					pass

	# email part
	email_list = []

	'''
	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE a.WWID=b.PIFLeadWWID AND b.BoardID = %s"
	val = (board_id,)
	piflist = execute_query(query,val)
	for i in range(len(piflist)):
		eid = piflist[0][1]
		email_list.append(eid)

	# all pif leads
	email_list += get_pif_leads_email_id_by_board_id(boardid=board_id)
	'''

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val = ('yes',)
	admin_email = execute_query(sql,val)
	for i in admin_email:
		email_list.append(i[0])

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (board_id,)
	sku_plat = execute_query(sql,val)


	board_filename = ''
	schematics_filename = ''
	stackup_filename = ''
	lenght_report_filename = ''
	other_filesname = ''
	reupload_comments = ''
	is_recent_upload = False

	sql = "SELECT a.BoardID,IFNULL(a.BoardFileName,''),IFNULL(a.SchematicsName,''),IFNULL(a.StackupFileName,''),IFNULL(a.LengthReportFileName,''),IFNULL(a.OthersFileName,''),IFNULL(b.Comments,''),IFNULL(a.IsRecentUpload,'') FROM UploadDesignFiles a LEFT JOIN UploadReSubmit b ON a.BoardID = b.BoardID WHERE a.BoardID = %s"
	val = (board_id,)
	rs_design_files = execute_query(sql,val)

	if rs_design_files != ():
		board_filename = rs_design_files[0][1]
		schematics_filename = rs_design_files[0][2]
		stackup_filename = rs_design_files[0][3]
		lenght_report_filename = rs_design_files[0][4]
		other_filesname = rs_design_files[0][5]
		reupload_comments = rs_design_files[0][6]

		if rs_design_files[0][7] != '':
			is_recent_upload = True

		# reset the flag, for next email trigger
		sql = "UPDATE UploadDesignFiles SET IsRecentUpload = %s WHERE BoardID = %s"
		val = (None,board_id)
		execute_query(sql, val)

	name = session.get('username')

	message = ''

	sql = "SELECT ScheduleStatusID FROM ScheduleTable WHERE BoardID = %s"
	val = (board_id,)
	statusid = execute_query(sql,val)[0][0]

	is_valid_for_mail = False
	all_category_lead_mail_enabled = False
	all_interface_mail_enabled = False
	new_interface_mail_enabled = False
	user_part_design_mail_enabled = False


	# yet to kickstart status - Design
	if(statusid ==3):
		# file upload + new interface
		if (new_comps != '') and is_recent_upload:

			subject = "[ID:" + board_id +"] Design collaterals submitted for review"
			message = ''' 	<b>Design Name: </b>'''+ board_name+ '''<br>
							Design Collaterals submitted by '''+name+''' <br><br>
							<b><u>Details of the uploaded collaterals:</u></b><br>
						  	<b>Board File: </b>'''+ str(board_filename) + '''<br>
						  	<b>Schematics: </b>'''+ str(schematics_filename) + '''<br>
						  	<b>Stackup: </b>'''+ str(stackup_filename) + '''<br>
						  	<b>Length Report: </b>'''+ str(lenght_report_filename) + '''<br>
						  	<b>Others: </b>'''+ str(other_filesname) + '''<br><br>
							Thanks,<br>
							ERAM.
				'''
			is_valid_for_mail = True
			user_part_design_mail_enabled = True

	
	# Design On-Going status
	if(statusid == 2):

		# file upload + new interface
		if (new_comps != '') and is_recent_upload:

			subject = "[ID:" + board_id +"] New interface and files added for review"
			message = ''' 	<b>Design Name: </b>'''+ board_name+ '''<br>
							New Interface and files added for review by '''+name+''' <br><br>
							<b><u>New Interface added to the design:</u></b>'''+new_comps+'''<br><br>
							<b><u>Details of the uploaded collaterals:</u></b><br>
						  	<b>Board File: </b>'''+ str(board_filename) + '''<br>
						  	<b>Schematics: </b>'''+ str(schematics_filename) + '''<br>
						  	<b>Stackup: </b>'''+ str(stackup_filename) + '''<br>
						  	<b>Length Report: </b>'''+ str(lenght_report_filename) + '''<br>
						  	<b>Others: </b>'''+ str(other_filesname) + '''<br>
							<b>Comments: </b><br>'''+str(reupload_comments)+'''<br><br>
							Thanks,<br>
							ERAM.
				'''
			is_valid_for_mail = True
			all_category_lead_mail_enabled = True
			all_interface_mail_enabled = True


		# only new interface addition
		elif new_comps != '':
			subject = "[ID:"+board_id +"] New interface added for review"
			message = ''' 	<b>Design Name: </b>'''+ board_name+ '''<br>
							New Interface added for review by '''+name+''' <br><br>
							<b><u>New Interface added to the design:</u></b>'''+new_comps+'''<br><br>
						  	<b><u>Details of the uploaded collaterals: </u></b><br>
						  	<b>Board File: </b>'''+ str(board_filename) + '''<br>
						  	<b>Schematics: </b>'''+ str(schematics_filename) + '''<br>
						  	<b>Stackup: </b>'''+ str(stackup_filename) + '''<br>
						  	<b>Length Report: </b>'''+ str(lenght_report_filename) + '''<br>
						  	<b>Others: </b>'''+ str(other_filesname) + '''<br><br>
						  	Thanks,<br>
						  	ERAM.
			'''

			is_valid_for_mail = True
			new_interface_mail_enabled = True

		# only file uploads
		elif is_recent_upload:
			subject = "[ID:"+board_id +"] Design collaterals re-submitted for review"
			message = ''' 	<b>Design Name: </b>'''+ board_name+ '''<br>
							Design Collaterals submitted by '''+name+''' <br><br>
							<b><u>Details of the uploaded collaterals:</u></b><br>
						  	<b>Board File: </b>'''+ str(board_filename) + '''<br>
						  	<b>Schematics: </b>'''+ str(schematics_filename) + '''<br>
						  	<b>Stackup: </b>'''+ str(stackup_filename) + '''<br>
						  	<b>Length Report: </b>'''+ str(lenght_report_filename) + '''<br>
						  	<b>Others: </b>'''+ str(other_filesname) + '''<br><br>
						  	Thanks,<br>
						  	ERAM.
			'''

			is_valid_for_mail = True
			all_category_lead_mail_enabled = True
			all_interface_mail_enabled = True


	if user_part_design_mail_enabled:

		query = "SELECT a.EmailID FROM RequestAccess a, BoardDetails b WHERE a.WWID=b.DesignLeadWWID and b.BoardID=%s"
		val = (board_id,)
		deslead = execute_query(query, val)
		email_list.append(deslead[0][0])

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.Username=b.DesignManagerWWID and b.BoardID AND b.BoardID = %s) LIMIT 1"
		val = (board_id,)
		designmanager = execute_query(query,val)
		if designmanager != ():
			email_list.append(designmanager[0][1])

		query = "SELECT a.EmailID FROM RequestAccess a, BoardDetails b WHERE a.WWID=b.CADLeadWWID and b.BoardID=%s"
		val = (board_id,)
		cad = execute_query(query, val)
		email_list.append(cad[0][0])

	if all_category_lead_mail_enabled:

		sql = "SELECT EmailID FROM HomeTable WHERE RoleID = %s"
		val=(14,)
		admin_list1 = execute_query(sql,val)				

		for j in admin_list1:
			email_list.append(j[0])

		query = "SELECT a.EmailID FROM RequestAccess a, BoardDetails b WHERE a.WWID=b.DesignLeadWWID and b.BoardID=%s"
		val = (board_id,)
		deslead = execute_query(query, val)
		email_list.append(deslead[0][0])

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.Username=b.DesignManagerWWID and b.BoardID AND b.BoardID = %s) LIMIT 1"
		val = (board_id,)
		designmanager = execute_query(query,val)
		if designmanager != ():
			email_list.append(designmanager[0][1])

		query = "SELECT a.EmailID FROM RequestAccess a, BoardDetails b WHERE a.WWID=b.CADLeadWWID and b.BoardID=%s"
		val = (board_id,)
		cad = execute_query(query, val)
		email_list.append(cad[0][0])

		# category leads
		catlead_sec_mail = []
		#sql="SELECT a.CategoryName,b.EmailID from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2,BoardReviewDesigner B1 WHERE cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PLatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID AND C2.ComponentID = B1.ComponentID ORDER BY cr.ComponentID"
		sql="SELECT a.CategoryName,b.EmailID,C3.CategoryLeadWWID1 from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2,BoardReviewDesigner B1 WHERE cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PLatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID AND C2.ComponentID = B1.ComponentID AND B1.BoardID = %s ORDER BY cr.ComponentID"
		val=(sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],board_id)
		catlead=execute_query(sql,val)
		if(catlead != ()):
			for i in catlead:
				email_list.append(i[1])

				if i[2] is not None:
					if i[2] != []:
						cat_sec_wwid_list = i[2][1:-1].split(', ')

						for j in range(0,len(cat_sec_wwid_list)):
							like_user_wwid = '%' + str(cat_sec_wwid_list[j]) + '%'
							if cat_sec_wwid_list[j] not in ['99999999',99999999]:
								sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
								val=(like_user_wwid,)
								catlead_sec_mail_rs = execute_query(sql,val)

								if catlead_sec_mail_rs != ():
									email_list.append(catlead_sec_mail_rs[0][0])

	if all_interface_mail_enabled:
		query = "SELECT a.EmailID FROM RequestAccess a, BoardDetails b WHERE a.WWID=b.DesignLeadWWID and b.BoardID=%s"
		val = (board_id,)
		deslead = execute_query(query, val)
		email_list.append(deslead[0][0])

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.Username=b.DesignManagerWWID and b.BoardID AND b.BoardID = %s) LIMIT 1"
		val = (board_id,)
		designmanager = execute_query(query,val)
		if designmanager != ():
			email_list.append(designmanager[0][1])

		query = "SELECT a.EmailID FROM RequestAccess a, BoardDetails b WHERE a.WWID=b.CADLeadWWID and b.BoardID=%s"
		val = (board_id,)
		cad = execute_query(query, val)
		email_list.append(cad[0][0])
		
		sql = "SELECT B1.ComponentID,C1.ComponentName FROM BoardReviewDesigner B1,ComponentType C1 WHERE B1.BoardID = %s AND B1.ComponentID = C1.ComponentID"	
		val = (board_id,)
		compids = execute_query(sql,val)	

		for q in compids:
			sql = "SELECT H1.EmailID FROM HomeTable H1, ComponentDesign C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
			val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
			primary_rev = execute_query(sql,val)

			sql = "SELECT H1.EmailID FROM HomeTable H1, ComponentReview C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
			val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
			primary_des = execute_query(sql,val)

			for j in primary_rev:
				email_list.append(j[0])
				
			for j in primary_des:
				email_list.append(j[0])
				
			
		sql = "select distinct SecondaryWWID from ComponentReview C2, BoardReviewDesigner B1, CategoryLeadTable C3,ComponentType C4 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s  AND C2.CategoryID = C3.CategoryID AND C3.SKUID = %s AND C3.PlatformID = %s AND  C3.MemTypeID = %s AND C3.DesignTypeID = %s "
		val = (board_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		sec_ele_wwid = execute_query(sql,val)

		sec_wwid = []

		if sec_ele_wwid != ():
			for i in sec_ele_wwid:
				if i[0] is not None:
					if (i[0] != []) and (i[0] != ['']):
						sec_wwid = i[0][1:-1].split(', ')

						for j in range(0,len(sec_wwid)):
							like_user_wwid = '%' + str(sec_wwid[j]) + '%'
							if sec_wwid[j] not in ['99999999',99999999]:
								sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
								val=(like_user_wwid,)
								email_id_rs = execute_query(sql,val)

								if email_id_rs != ():
									email_list.append(email_id_rs[0][0])

		sql = "select distinct SecondaryWWID FROM ComponentDesign C2,BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s"
		val = (board_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		sec_des_wwid = execute_query(sql,val)

		#ele_wwid = []
		des_wwid = []
		if sec_des_wwid != ():
			for i in sec_des_wwid:
				if i[0] is not None:
					if (i[0] != []) and (i[0] != ['']):
						des_wwid = i[0][1:-1].split(', ')

						for j in range(0,len(des_wwid)):
							like_user_wwid = '%' + str(des_wwid[j]) + '%'
							if des_wwid[j] not in ['99999999',99999999]:
								sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
								val=(like_user_wwid,)
								email_id_rs = execute_query(sql,val)

								if email_id_rs != ():
									email_list.append(email_id_rs[0][0])

	new_comps_id_list = tuple(new_comps_id_list)

	if new_interface_mail_enabled:
		query = "SELECT a.EmailID FROM RequestAccess a, BoardDetails b WHERE a.WWID=b.DesignLeadWWID and b.BoardID=%s"
		val = (board_id,)
		deslead = execute_query(query, val)
		email_list.append(deslead[0][0])

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.Username=b.DesignManagerWWID and b.BoardID AND b.BoardID = %s) LIMIT 1"
		val = (board_id,)
		designmanager = execute_query(query,val)
		if designmanager != ():
			email_list.append(designmanager[0][1])

		query = "SELECT a.EmailID FROM RequestAccess a, BoardDetails b WHERE a.WWID=b.CADLeadWWID and b.BoardID=%s"
		val = (board_id,)
		cad = execute_query(query, val)
		email_list.append(cad[0][0])

		#for q in new_comps_id_list:
		sql = "SELECT H1.EmailID FROM HomeTable H1, ComponentDesign C2 WHERE C2.ComponentID IN %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
		val = (new_comps_id_list,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		primary_rev = execute_query(sql,val)

		sql = "SELECT H1.EmailID FROM HomeTable H1, ComponentReview C2 WHERE C2.ComponentID IN %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
		val = (new_comps_id_list,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		primary_des = execute_query(sql,val)

		for j in primary_rev:
			email_list.append(j[0])
			
		for j in primary_des:
			email_list.append(j[0])
			
		
		sql = "SELECT distinct SecondaryWWID from ComponentReview C2, BoardReviewDesigner B1, CategoryLeadTable C3,ComponentType C4 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID IN %s AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s  AND C2.CategoryID = C3.CategoryID AND C3.SKUID = %s AND C3.PlatformID = %s AND  C3.MemTypeID = %s AND C3.DesignTypeID = %s "
		val = (board_id,new_comps_id_list,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		sec_ele_wwid = execute_query(sql,val)

		sec_wwid = []

		if sec_ele_wwid != ():
			for i in sec_ele_wwid:
				if i[0] is not None:
					if (i[0] != []) and (i[0] != ['']):
						sec_wwid = i[0][1:-1].split(', ')

						for j in range(0,len(sec_wwid)):
							like_user_wwid = '%' + str(sec_wwid[j]) + '%'
							if sec_wwid[j] not in ['99999999',99999999]:
								sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
								val=(like_user_wwid,)
								email_id_rs = execute_query(sql,val)

								if email_id_rs != ():
									email_list.append(email_id_rs[0][0])

		sql = "SELECT distinct SecondaryWWID FROM ComponentDesign C2,BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID IN %s AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s"
		val = (board_id,new_comps_id_list,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		sec_des_wwid = execute_query(sql,val)

		#ele_wwid = []
		des_wwid = []
		if sec_des_wwid != ():
			for i in sec_des_wwid:
				if i[0] is not None:
					if (i[0] != []) and (i[0] != ['']):
						des_wwid = i[0][1:-1].split(', ')

						for j in range(0,len(des_wwid)):
							like_user_wwid = '%' + str(des_wwid[j]) + '%'
							if des_wwid[j] not in ['99999999',99999999]:
								sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
								val=(like_user_wwid,)
								email_id_rs = execute_query(sql,val)

								if email_id_rs != ():
									email_list.append(email_id_rs[0][0])

		catlead_sec_mail = []
		sql="SELECT a.CategoryName,b.EmailID,C3.CategoryLeadWWID1 from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2,BoardReviewDesigner B1 WHERE cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PLatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID AND C2.ComponentID = B1.ComponentID AND B1.BoardID = %s AND  B1.ComponentID IN %s ORDER BY cr.ComponentID"
		val=(sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],board_id,new_comps_id_list)
		catlead=execute_query(sql,val)
		if(catlead != ()):
			for i in catlead:
				email_list.append(i[1])

				if i[2] is not None:
					if i[2] != []:
						cat_sec_wwid_list = i[2][1:-1].split(', ')

						for j in range(0,len(cat_sec_wwid_list)):
							like_user_wwid = '%' + str(cat_sec_wwid_list[j]) + '%'
							if cat_sec_wwid_list[j] not in ['99999999',99999999]:
								sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
								val=(like_user_wwid,)
								catlead_sec_mail_rs = execute_query(sql,val)

								if catlead_sec_mail_rs != ():
									email_list.append(catlead_sec_mail_rs[0][0])

	if is_valid_for_mail:
		email_list = sorted(set(email_list), reverse=True)	
		for i in email_list:
			send_mail(i,subject,message,email_list)

	return design_files(boardid=int(board_id),comp_selected_list=comp_selected_list,my_designs="All",my_interfaces="All")

@app.route("/design_files_submit",methods = ['POST', 'GET'])
def design_files_submit():
	
	wwid=session.get('wwid')
	username = session.get('username')

	data = {}
	data = dict(request.form)

	board_id = data['board_id']
	comp_list = eval(data['comp_select[]'])

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	# log table
	try:
		log_notes = 'User has submitted Design Files for Review for Design ID: '+str(board_id)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Design Files',board_id,0,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	query = "SELECT BoardFileID,BoardFileName,SchematicsFileID,SchematicsName,StackupFileID,StackupFileName,LengthReportFileID,LengthReportFileName,OthersFileID,OthersFileName FROM UploadDesignFiles WHERE BoardID = %s"
	val=(board_id,)
	file_details_rs=execute_query(query,val)

	if file_details_rs !=():
		board_file_id = copy.deepcopy(file_details_rs[0][0])
		board_filename = copy.deepcopy(file_details_rs[0][1])
		schematics_file_id = copy.deepcopy(file_details_rs[0][2])
		schematics_filename = copy.deepcopy(file_details_rs[0][3])
		stackup_file_id = copy.deepcopy(file_details_rs[0][4])
		stackup_filename = copy.deepcopy(file_details_rs[0][5])
		lenght_report_file_id = copy.deepcopy(file_details_rs[0][6])
		lenght_report_filename = copy.deepcopy(file_details_rs[0][7])
		other_files_id = copy.deepcopy(file_details_rs[0][8])
		other_filesname = copy.deepcopy(file_details_rs[0][9])
	else:
		board_file_id = None
		board_filename = None
		schematics_file_id = None
		schematics_filename = None
		stackup_file_id = None
		stackup_filename = None
		lenght_report_file_id = None
		lenght_report_filename = None
		other_files_id = None
		other_filesname = None

	is_valid_files = False

	if request.files['board_file'].filename != '':

		if board_filename is None:
			board_filename = ""

		'''
		#if ((board_filename is not None) and (board_filename != request.files['board_file'].filename)):
		if board_filename != request.files['board_file'].filename:
			query = "SELECT a.BoardID,a.ComponentID,a.CommentID FROM BoardReview a LEFT JOIN ScheduleTableComponent b ON a.BoardID=b.BoardID AND a.ComponentID=b.ComponentID WHERE a.BoardID = %s AND (b.ScheduleStatusID = %s OR b.ScheduleStatusID IS NULL) and a.BoardFileName = %s"
			val=(board_id,3,board_filename)
			file_name_rs_temp=execute_query(query,val)

			if file_name_rs_temp != ():
				for row in file_name_rs_temp:
					sql = "UPDATE BoardReview SET BoardFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (request.files['board_file'].filename,row[0],row[1],row[2])
					execute_query(sql,val)
		'''
		#if ((board_filename is not None) and (board_filename != request.files['board_file'].filename)):
		if board_filename != request.files['board_file'].filename:
			query = "SELECT a.BoardID,a.ComponentID,a.CommentID FROM BoardReview a LEFT JOIN ScheduleTableComponent b ON a.BoardID=b.BoardID AND a.ComponentID=b.ComponentID WHERE a.BoardID = %s AND (b.ScheduleStatusID = %s OR b.ScheduleStatusID IS NULL) and a.BoardFileName = %s and a.DesignDocument = %s"
			val=(board_id,3,board_filename,"Board File")
			file_name_rs_temp=execute_query(query,val)

			if file_name_rs_temp != ():
				for row in file_name_rs_temp:
					sql = "UPDATE BoardReview SET BoardFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (request.files['board_file'].filename,row[0],row[1],row[2])
					execute_query(sql,val)

		board_file = request.files['board_file']
		board_filename = secure_filename(request.files['board_file'].filename)

		# attaching file for litepi process
		file_response = litepi_file_process(boardid=board_id,file_name=board_filename,file_upload=board_file)
		print("file_response: ",file_response)

		board_file.stream.seek(0)	# reset file stream pointer
		board_file_read = board_file.read()	# read bytes from the file to upload in DB

		sql = "INSERT INTO UploadFileStorage VALUES (%s,%s,%s)"
		val = (None,board_id,board_file_read)
		execute_query(sql,val)

		sql = "SELECT LAST_INSERT_ID()"
		board_file_id = execute_query_sql(sql)[0][0]

		sql = "INSERT INTO UploadFileStorageMisc VALUES (%s,%s,%s,%s)"
		val = (board_file_id,board_id,t,wwid)
		execute_query(sql,val)

		is_valid_files = True


	if request.files['schematics_file'].filename != '':
		
		if schematics_filename is None:
			schematics_filename = ""

		'''
		#if ((schematics_filename is not None) and (schematics_filename != request.files['schematics_file'].filename)):
		if schematics_filename != request.files['schematics_file'].filename:
			query = "SELECT a.BoardID,a.ComponentID,a.CommentID FROM BoardReview a LEFT JOIN ScheduleTableComponent b ON a.BoardID=b.BoardID AND a.ComponentID=b.ComponentID WHERE a.BoardID = %s AND (b.ScheduleStatusID = %s OR b.ScheduleStatusID IS NULL) and a.BoardFileName = %s"
			val=(board_id,3,schematics_filename)
			file_name_rs_temp=execute_query(query,val)

			if file_name_rs_temp != ():
				for row in file_name_rs_temp:
					sql = "UPDATE BoardReview SET BoardFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (request.files['schematics_file'].filename,row[0],row[1],row[2])
					execute_query(sql,val)				
		'''
		if schematics_filename != request.files['schematics_file'].filename:
			query = "SELECT a.BoardID,a.ComponentID,a.CommentID FROM BoardReview a LEFT JOIN ScheduleTableComponent b ON a.BoardID=b.BoardID AND a.ComponentID=b.ComponentID WHERE a.BoardID = %s AND (b.ScheduleStatusID = %s OR b.ScheduleStatusID IS NULL) and a.BoardFileName = %s and a.DesignDocument = %s"
			val=(board_id,3,schematics_filename,"Schematics File")
			file_name_rs_temp=execute_query(query,val)

			if file_name_rs_temp != ():
				for row in file_name_rs_temp:
					sql = "UPDATE BoardReview SET BoardFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (request.files['schematics_file'].filename,row[0],row[1],row[2])
					execute_query(sql,val)				

		schematics_file = request.files['schematics_file'].read()
		schematics_filename = request.files['schematics_file'].filename

		sql = "INSERT INTO UploadFileStorage VALUES (%s,%s,%s)"
		val = (None,board_id,schematics_file)
		execute_query(sql,val)

		sql = "SELECT LAST_INSERT_ID()"
		schematics_file_id = execute_query_sql(sql)[0][0]

		sql = "INSERT INTO UploadFileStorageMisc VALUES (%s,%s,%s,%s)"
		val = (schematics_file_id,board_id,t,wwid)
		execute_query(sql,val)

		is_valid_files = True
		
	if request.files['stackup_file'].filename != '':

		if stackup_filename is None:
			stackup_filename = ""

		'''
		#if ((stackup_filename is not None) and (stackup_filename != request.files['stackup_file'].filename)):
		if stackup_filename != request.files['stackup_file'].filename:
			query = "SELECT a.BoardID,a.ComponentID,a.CommentID FROM BoardReview a LEFT JOIN ScheduleTableComponent b ON a.BoardID=b.BoardID AND a.ComponentID=b.ComponentID WHERE a.BoardID = %s AND (b.ScheduleStatusID = %s OR b.ScheduleStatusID IS NULL) and a.BoardFileName = %s"
			val=(board_id,3,stackup_filename)
			file_name_rs_temp=execute_query(query,val)

			if file_name_rs_temp != ():
				for row in file_name_rs_temp:
					sql = "UPDATE BoardReview SET BoardFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (request.files['stackup_file'].filename,row[0],row[1],row[2])
					execute_query(sql,val)		
		'''		

		#if ((stackup_filename is not None) and (stackup_filename != request.files['stackup_file'].filename)):
		if stackup_filename != request.files['stackup_file'].filename:
			query = "SELECT a.BoardID,a.ComponentID,a.CommentID FROM BoardReview a LEFT JOIN ScheduleTableComponent b ON a.BoardID=b.BoardID AND a.ComponentID=b.ComponentID WHERE a.BoardID = %s AND (b.ScheduleStatusID = %s OR b.ScheduleStatusID IS NULL) and a.BoardFileName = %s and a.DesignDocument = %s"
			val=(board_id,3,stackup_filename,"Stackup File")
			file_name_rs_temp=execute_query(query,val)

			if file_name_rs_temp != ():
				for row in file_name_rs_temp:
					sql = "UPDATE BoardReview SET BoardFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (request.files['stackup_file'].filename,row[0],row[1],row[2])
					execute_query(sql,val)				

		stackup_file = request.files['stackup_file'].read()
		stackup_filename = request.files['stackup_file'].filename

		sql = "INSERT INTO UploadFileStorage VALUES (%s,%s,%s)"
		val = (None,board_id,stackup_file)
		execute_query(sql,val)

		sql = "SELECT LAST_INSERT_ID()"
		stackup_file_id = execute_query_sql(sql)[0][0]

		sql = "INSERT INTO UploadFileStorageMisc VALUES (%s,%s,%s,%s)"
		val = (stackup_file_id,board_id,t,wwid)
		execute_query(sql,val)

		is_valid_files = True

	if request.files['lenght_report_file'].filename != '':

		if lenght_report_filename is None:
			lenght_report_filename = ""

		'''
		#if ((lenght_report_filename is not None) and (lenght_report_filename != request.files['lenght_report_file'].filename)):
		if lenght_report_filename != request.files['lenght_report_file'].filename:
			query = "SELECT a.BoardID,a.ComponentID,a.CommentID FROM BoardReview a LEFT JOIN ScheduleTableComponent b ON a.BoardID=b.BoardID AND a.ComponentID=b.ComponentID WHERE a.BoardID = %s AND (b.ScheduleStatusID = %s OR b.ScheduleStatusID IS NULL) and a.BoardFileName = %s"
			val=(board_id,3,lenght_report_filename)
			file_name_rs_temp=execute_query(query,val)

			if file_name_rs_temp != ():
				for row in file_name_rs_temp:
					sql = "UPDATE BoardReview SET BoardFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (request.files['lenght_report_file'].filename,row[0],row[1],row[2])
					execute_query(sql,val)
		'''
		#if ((lenght_report_filename is not None) and (lenght_report_filename != request.files['lenght_report_file'].filename)):
		if lenght_report_filename != request.files['lenght_report_file'].filename:
			query = "SELECT a.BoardID,a.ComponentID,a.CommentID FROM BoardReview a LEFT JOIN ScheduleTableComponent b ON a.BoardID=b.BoardID AND a.ComponentID=b.ComponentID WHERE a.BoardID = %s AND (b.ScheduleStatusID = %s OR b.ScheduleStatusID IS NULL) and a.BoardFileName = %s and a.DesignDocument = %s"
			val=(board_id,3,lenght_report_filename,"Lenght Report File")
			file_name_rs_temp=execute_query(query,val)

			if file_name_rs_temp != ():
				for row in file_name_rs_temp:
					sql = "UPDATE BoardReview SET BoardFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (request.files['lenght_report_file'].filename,row[0],row[1],row[2])
					execute_query(sql,val)

		lenght_report_file = request.files['lenght_report_file'].read()
		lenght_report_filename = request.files['lenght_report_file'].filename

		sql = "INSERT INTO UploadFileStorage VALUES (%s,%s,%s)"
		val = (None,board_id,lenght_report_file)
		execute_query(sql,val)

		sql = "SELECT LAST_INSERT_ID()"
		lenght_report_file_id = execute_query_sql(sql)[0][0]

		sql = "INSERT INTO UploadFileStorageMisc VALUES (%s,%s,%s,%s)"
		val = (lenght_report_file_id,board_id,t,wwid)
		execute_query(sql,val)

		is_valid_files = True

	if request.files['other_files'].filename != '':

		if other_filesname is None:
			other_filesname = ""
			
		'''
		#if ((other_filesname is not None) and (other_filesname != request.files['other_files'].filename)):
		if other_filesname != request.files['other_files'].filename:
			query = "SELECT a.BoardID,a.ComponentID,a.CommentID FROM BoardReview a LEFT JOIN ScheduleTableComponent b ON a.BoardID=b.BoardID AND a.ComponentID=b.ComponentID WHERE a.BoardID = %s AND (b.ScheduleStatusID = %s OR b.ScheduleStatusID IS NULL) and a.BoardFileName = %s"
			val=(board_id,3,other_filesname)
			file_name_rs_temp=execute_query(query,val)

			if file_name_rs_temp != ():
				for row in file_name_rs_temp:
					sql = "UPDATE BoardReview SET BoardFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (request.files['other_files'].filename,row[0],row[1],row[2])
					execute_query(sql,val)
		'''
			
		#if ((other_filesname is not None) and (other_filesname != request.files['other_files'].filename)):
		if other_filesname != request.files['other_files'].filename:
			query = "SELECT a.BoardID,a.ComponentID,a.CommentID FROM BoardReview a LEFT JOIN ScheduleTableComponent b ON a.BoardID=b.BoardID AND a.ComponentID=b.ComponentID WHERE a.BoardID = %s AND (b.ScheduleStatusID = %s OR b.ScheduleStatusID IS NULL) and a.BoardFileName = %s and a.DesignDocument = %s"
			val=(board_id,3,other_filesname,"Others File")
			file_name_rs_temp=execute_query(query,val)

			if file_name_rs_temp != ():
				for row in file_name_rs_temp:
					sql = "UPDATE BoardReview SET BoardFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (request.files['other_files'].filename,row[0],row[1],row[2])
					execute_query(sql,val)				

		other_files = request.files['other_files'].read()
		other_filesname = request.files['other_files'].filename

		sql = "INSERT INTO UploadFileStorage VALUES (%s,%s,%s)"
		val = (None,board_id,other_files)
		execute_query(sql,val)

		sql = "SELECT LAST_INSERT_ID()"
		other_files_id = execute_query_sql(sql)[0][0]

		sql = "INSERT INTO UploadFileStorageMisc VALUES (%s,%s,%s,%s)"
		val = (other_files_id,board_id,t,wwid)
		execute_query(sql,val)

		is_valid_files = True

	# if no files are uploaded, then ignore all further update and email process
	if not is_valid_files:
		return design_files(boardid=int(board_id),comp_selected_list=comp_list,my_designs="All",my_interfaces="All")

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	filename = None
	file_id = None
	reviewfiles = None

	sql = "INSERT INTO UploadDesignFiles VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s) ON DUPLICATE KEY UPDATE FileName = %s, FileID = %s, ReviewFilenames = %s, WWID = %s, Insert_Time = %s, BoardFileID = %s, BoardFileName = %s, SchematicsFileID = %s, SchematicsName = %s, StackupFileID = %s, StackupFileName = %s, LengthReportFileID = %s, LengthReportFileName = %s, OthersFileID = %s, OthersFileName = %s, Count = Count + 1, IsRecentUpload = %s"
	val = (board_id,filename,file_id,reviewfiles,wwid,t,board_file_id,board_filename,schematics_file_id,schematics_filename,stackup_file_id,stackup_filename,lenght_report_file_id,lenght_report_filename,other_files_id,other_filesname,1,"yes",filename,file_id,reviewfiles,wwid,t,board_file_id,board_filename,schematics_file_id,schematics_filename,stackup_file_id,stackup_filename,lenght_report_file_id,lenght_report_filename,other_files_id,other_filesname,"yes")
	execute_query(sql,val)

	if 'reupload_comments' in data:
		
		resubmit_comments = data['reupload_comments']

		comments = str(resubmit_comments)+'<br>'+'<span style="color:grey; font-size: smaller;">Re-submitted by: '+str(username)+'</span>'
		comments_update = '<br><br>'+comments
		sql = "INSERT INTO UploadReSubmit VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE Comments = CONCAT(Comments,%s), Insert_Time = %s"
		val = (board_id,comments,t,comments_update,t)
		execute_query(sql,val)
	
	return design_files(boardid=int(board_id),comp_selected_list=comp_list,my_designs="All",my_interfaces="All")

@app.route("/get_design_comp",methods = ['POST', 'GET'])
def get_design_comp():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))

	boardid = copy.deepcopy(data['boardid'])
	my_interfaces = copy.deepcopy(data['my_interfaces'])

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = 'no'
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = 'yes'

	sql = "SELECT PIFLeadWWID From BoardDetails WHERE BoardID = %s"
	val  =(boardid,)
	pif = execute_query(sql,val)
	ispif = False
	if pif != ():
		if(str(pif[0][0]) == wwid ):
			ispif = True
	
	comp_status_list = []
	comp_list = []
	feedbacks = []
					
	if my_interfaces == "All":
		temp_data = []
		temp_data = get_all_interfaces(boardid=boardid)

	else:
		temp_data = []
		temp_data = get_my_interfaces(boardid=boardid)

	comp_list = temp_data[1]
	#comp_status_list = get_status_list_sorted(data_list=temp_data[0])
	comp_status_list = get_order_status_list(list=temp_data[0])

	final_result = [comp_status_list,comp_list]

	return jsonify(final_result)


@app.route("/delete_feedback_row",methods = ['POST', 'GET'])
def delete_feedback_row():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))

	board_id = copy.deepcopy(data['board_id'])
	comp_id = copy.deepcopy(data['comp_id'])
	comment_id = copy.deepcopy(data['comment_id'])

	sql = "DELETE FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
	val  =(board_id,comp_id,comment_id)
	result = execute_query(sql,val)

	# log table
	try:
		log_notes = 'User has Deleted Feedback for <br>Design ID: '+str(board_id)+'<br>Component Name: '+str(comp_id)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Delete Feedback',board_id,0,comp_id,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	return jsonify(result)

@app.route("/delete_child_feedback_row",methods = ['POST', 'GET'])
def delete_child_feedback_row():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))

	board_id = copy.deepcopy(data['board_id'])
	comp_id = copy.deepcopy(data['comp_id'])
	comment_id = copy.deepcopy(data['comment_id'])
	parent_comment_id = copy.deepcopy(data['parent_comment_id'])

	sql = "DELETE FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
	val  =(board_id,comp_id,comment_id)
	result = execute_query(sql,val)

	sql = "UPDATE BoardReview SET HasChild = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
	val  =(None,board_id,comp_id,parent_comment_id)
	result = execute_query(sql,val)

	# log table
	try:
		log_notes = 'User has Deleted Child Feedback for <br>Design ID: '+str(board_id)+'<br>Component Name: '+str(comp_id)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Delete Feedback',board_id,0,comp_id,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	return jsonify(result)


@app.route('/delete_comp_design_page',methods = ['POST', 'GET'])
def delete_comp_design_page():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))

	board_id = copy.deepcopy(data['boardid'])
	comp_id = copy.deepcopy(data['compid'])

	sql="DELETE FROM BoardReviewDesigner WHERE (BoardID=%s AND ComponentID=%s)"
	val=(board_id,comp_id)
	execute_query(sql, val)

	sql="DELETE FROM ScheduleTableComponent WHERE (BoardID=%s AND ComponentID=%s)"
	val=(board_id,comp_id)
	execute_query(sql, val)

	sql = "DELETE FROM BoardReview WHERE BoardID = %s AND ComponentID  =%s"
	val = (board_id,comp_id)
	execute_query(sql,val)

	sql = "DELETE FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID  =%s"
	val = (board_id,comp_id)
	execute_query(sql,val)

	# log table
	try:
		log_notes = 'User has Deleted Interface for <br>Design ID: '+str(board_id)+'<br>Component Name: '+str(comp_id)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Delete Interface',board_id,0,comp_id,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	return jsonify(True)

def get_listed_status_designs(my_designs_id=[]):

	wwid=session.get('wwid')
	username = session.get('username')

	bdetails = []
	ww_date = []

	if my_designs_id == []:
		my_designs_id = (0,1)
	else:
		my_designs_id = tuple(my_designs_id)

	# for Ongoing Designs
	# for boardid in my_designs_id:
	sql = "SELECT B.BoardName,B.BoardID,H.UserName,D2.ProposedStartDate,D2.ProposedEndDate,D2.BoardState,S1.ScheduleTypeName FROM BoardDetails B, HomeTable H, DesignCalendar D2, ScheduleStatusType S1, ScheduleTable S2 WHERE B.DesignLeadWWID = H.WWID AND B.BoardID = D2.BoardID AND S2.BoardID = B.BoardID AND S1.ScheduleID = S2.ScheduleStatusID AND B.BoardID IN %s AND S1.ScheduleID IN %s ORDER BY S1.ScheduleID,D2.ProposedEndDate"
	val = (my_designs_id,(2,6))
	blist = execute_query(sql,val)

	for row in blist:
		bdetails.append(row)

	#sql = "SELECT B.BoardName,B.BoardID,H.UserName,D2.ProposedStartDate,D2.ProposedEndDate,D2.BoardState,S1.ScheduleTypeName FROM BoardDetails B, HomeTable H, DesignCalendar D2, ScheduleStatusType S1, ScheduleTable S2 WHERE B.DesignLeadWWID = H.WWID AND B.BoardID = D2.BoardID AND S2.BoardID = B.BoardID AND S1.ScheduleID = S2.ScheduleStatusID AND B.BoardID IN %s AND S1.ScheduleID = %s ORDER BY FIELD(D2.BoardState,'Design Team Projection','ERAM Timeline Commit','Projected & No-Updates','Design Review In-Progress','Design Not signed-off','Design Signed-off'),D2.ProposedStartDate"
	sql = "SELECT B.BoardName,B.BoardID,H.UserName,D2.ProposedStartDate,D2.ProposedEndDate,D2.BoardState,S1.ScheduleTypeName FROM BoardDetails B, HomeTable H, DesignCalendar D2, ScheduleStatusType S1, ScheduleTable S2 WHERE B.DesignLeadWWID = H.WWID AND B.BoardID = D2.BoardID AND S2.BoardID = B.BoardID AND S1.ScheduleID = S2.ScheduleStatusID AND B.BoardID IN %s AND S1.ScheduleID = %s ORDER BY D2.ProposedStartDate"
	val = (my_designs_id,3)
	blist = execute_query(sql,val)

	for row in blist:
		bdetails.append(row)

	for row in bdetails:
		if row[6] == 'Ongoing':
			ww_date.append('Feedback Due Date: ' + get_work_week_fun_with_year(row[4]))

		elif row[6] == 'Yet_to_Kickstart':
			ww_date.append('Start Date: ' + get_work_week_fun_with_year(row[3]))

		elif row[6] == 'Signed-Off':
			ww_date.append('Signed-Off Date: ' + get_work_week_fun_with_year(row[4]))

		else:
			ww_date.append('End Date: ' + get_work_week_fun_with_year(row[4]))

	return bdetails,ww_date

@app.route("/feedbacks",methods = ['POST', 'GET'])
def feedbacks(boardid=None,comp_selected_list=[],my_designs="My",my_interfaces="My"):
	
	wwid=session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	if wwid is None:
		print("invalid sso login")
		session['target_page'] = 'feedbacks'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	is_admin = False
	if has_admin_access == "yes":
		is_admin = True

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	if boardid == None:
		if request.method == 'POST':
			boardid = request.form.get("boardid", type=int)
			comp_selected_list = request.form.getlist("comp_select")

			if comp_selected_list is None:
				comp_selected_list = []

			my_designs = request.form.get("my_designs")

			if my_designs is None:
				my_designs = "My"
			my_interfaces = request.form.get("my_interfaces")

			if my_interfaces is None:
				my_interfaces = "My"
		
	data = {}

	if boardid == None:
		boardid = 0

	if comp_selected_list == None:
		comp_selected_list = []


	sql = "SELECT AreaofIssue FROM AreaOfIssue ORDER BY AreaofIssue"
	area = execute_query_sql(sql)

	areas=[]
	for i in area:
		areas.append(i[0])

	sql = "SELECT * FROM BoardDetails WHERE BoardID = %s AND (DesignLeadWWID = %s OR CADLeadWWID = %s)"	
	val = (boardid,wwid,wwid)	
	rs_design_layout_lead = execute_query(sql,val)

	is_design_layout_lead = False
	if rs_design_layout_lead != ():
		is_design_layout_lead = True

	design_status_list = []
	design_list = []
	my_designs_id = []

	temp_data = []
	temp_data = get_my_designs_feedbacks()
	my_designs_all_checked = ""
	my_designs_my_checked = "checked"		

	for i in range(0,len(temp_data[1])):
		my_designs_id.append(temp_data[1][i][0])

	if my_designs == "All":
		temp_data = get_all_designs_feedbacks()
		my_designs_all_checked = "checked"
		my_designs_my_checked = ""

	#design_status_list = get_status_list_sorted(data_list=temp_data[0])
	design_status_list = get_order_status_list(list=temp_data[0])

	design_list = temp_data[1]

	comp_status_list = []
	comp_list = []
	my_components_id = []

	temp_data = []

	# for design & Layout Lead, Managers - both My interface and All interface should be same and have edit access, as we dont have any mapping for design/layout lead and managers at backend properly
	if is_design_layout_lead:
		temp_data = get_all_interfaces_feedbacks(boardid=boardid)
	else:
		temp_data = get_my_interfaces_feedbacks(boardid=boardid)

	my_interfaces_all_checked = ""
	my_interfaces_my_checked = "checked"	

	for i in range(0,len(temp_data[1])):
		my_components_id.append(temp_data[1][i][0])

	if my_interfaces == "All":
		temp_data = []
		temp_data = get_all_interfaces_feedbacks(boardid=boardid)
		my_interfaces_all_checked = "checked"
		my_interfaces_my_checked = ""


	#comp_status_list = get_status_list_sorted(data_list=temp_data[0])
	comp_status_list = get_order_status_list(list=temp_data[0])

	comp_list = temp_data[1]

	# for listing my designs with dates
	boards = []
	ww_date = []
	boards,ww_date = get_listed_status_designs(my_designs_id=my_designs_id)

	board_name = ''
	is_rev0p6_design = False
	is_rev1p0_design = True

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID,BoardName,ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	if sku_plat != ():
		board_name = sku_plat[0][4]

		if sku_plat[0][5] in [1,'1']:
			is_rev0p6_design = True
			is_rev1p0_design = False

	try:
		sql = "SELECT ScheduleStatusID FROM ScheduleTable WHERE BoardID = %s"
		val = (boardid,)
		board_status = execute_query(sql,val)[0][0]
	except:
		board_status = 0

	# to get Intrfaces level details
	for compid in comp_selected_list:
		data = get_feedbacks_data_page(data=data,boardid=boardid,compid=compid,complist=comp_list,sku_plat=sku_plat,board_status=board_status,my_designs_id=my_designs_id,my_components_id=my_components_id)

	# edit permission check
	file_upload_edit_enabled = False

	# design file details
	sql = "SELECT IFNULL(a.BoardFileName,''),IFNULL(a.SchematicsName,''),IFNULL(a.StackupFileName,''),IFNULL(a.LengthReportFileName,''),IFNULL(a.OthersFileName,''),IFNULL(b.Comments,''),IFNULL(c.UserName,''),IFNULL(a.Insert_Time,'') FROM UploadSignoffLatestFiles a LEFT JOIN UploadReSubmit b ON a.BoardID = b.BoardID LEFT JOIN HomeTable c on c.WWID = a.WWID WHERE a.BoardID = %s"
	val = (boardid,)
	file_details = execute_query(sql,val)

	design_document_name_list = []

	if file_details != ():
		file_details_available = True
		file_details_colspan = 4

		design_document_name_list.append([file_details[0][0],"Board File"])
		design_document_name_list.append([file_details[0][1],"Schematics File"])
		design_document_name_list.append([file_details[0][2],"Stackup File"])
		design_document_name_list.append([file_details[0][3],"Lenght Report File"])
		design_document_name_list.append([file_details[0][4],"Others File"])

	else:
		file_details_available = False
		file_details_colspan = 2+1

		# initializing if there are no records
		file_details = (("","","","",""))

	design_document_name_list = [row for row in design_document_name_list if row[0] != '']

	sql = "SELECT a.ScheduleStatusID FROM ScheduleTable a WHERE a.BoardID = %s AND a.ScheduleStatusID IN (2,3,6)"
	val = (boardid,)
	design_status_check = execute_query(sql,val)

	if design_status_check != ():
		if has_admin_access == "yes":
			file_upload_edit_enabled = True

		if ((is_design_owner) or (is_layout_owner)):
			if boardid in my_designs_id:
				file_upload_edit_enabled = True

	# to keep view mode for these 2 design
	if boardid in [60,61,'60','61']:
		file_upload_edit_enabled = False

	if not file_upload_edit_enabled:
		file_details_colspan -= 1

	if boardid == 0:
		design_list_table_show = "block"
		up_arrow_btn = "block"
		down_arrow_btn = "none"
	else:
		design_list_table_show = "none"
		up_arrow_btn = "none"
		down_arrow_btn = "block"

	return render("feedbacks.html",is_rev0p6_design=is_rev0p6_design,is_rev1p0_design=is_rev1p0_design,user_role_name=user_role_name,wwid=wwid,up_arrow_btn=up_arrow_btn,down_arrow_btn=down_arrow_btn,design_list_table_show=design_list_table_show,boards=boards,region_name=region_name,board_name=board_name,ww_date=ww_date,data=data,boardid=boardid,is_admin=is_admin,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,is_elec_owner=is_elec_owner,design_document_name_list=design_document_name_list,my_designs_all_checked=my_designs_all_checked,my_designs_my_checked=my_designs_my_checked,my_interfaces_all_checked=my_interfaces_all_checked,my_interfaces_my_checked=my_interfaces_my_checked,file_upload_edit_enabled=file_upload_edit_enabled,file_details=file_details,file_details_colspan=file_details_colspan,file_details_available=file_details_available,areas=areas,comp_selected_list=comp_selected_list,username = username,design_status_list=design_status_list,design_list=design_list,comp_status_list=comp_status_list,comp_list=comp_list)

@app.route("/get_designs_ajax_feedbacks",methods = ['POST', 'GET'])
def get_designs_ajax_feedbacks():

	wwid=session.get('wwid')
	username = session.get('username')

	data = request.form.get("data")

	result = {}

	design_status_list = []
	design_list = []

	if data == "All":
		temp_data = []
		temp_data = get_all_designs_feedbacks()

	else:
		temp_data = []
		temp_data = get_my_designs_feedbacks()

	#result["design_status_list"] = json.dumps(get_status_list_sorted(data_list=temp_data[0]))
	result["design_status_list"] = json.dumps(get_order_status_list(list=temp_data[0]))
	result["design_list"] = json.dumps(temp_data[1])

	return jsonify(result)

def get_all_designs_owners():

	wwid=session.get('wwid')
	username = session.get('username')

	design_list = []
	design_status_list = []

	sql = "SELECT DISTINCT c.ScheduleTypeName FROM BoardDetails a, ScheduleTable b,ScheduleStatusType c WHERE a.BoardID=b.BoardID AND b.ScheduleStatusID=c.ScheduleID ORDER BY FIELD(b.ScheduleStatusID,2,6,3,7,5,1,4),a.BoardID DESC"
	result = execute_query_sql(sql)

	if result != ():
		for i in range(0,len(result)):
			design_status_list.append(result[i][0])


	sql = "SELECT a.BoardID,a.BoardName,c.ScheduleTypeName,b.ScheduleStatusID FROM BoardDetails a, ScheduleTable b,ScheduleStatusType c WHERE a.BoardID=b.BoardID AND b.ScheduleStatusID=c.ScheduleID ORDER BY FIELD(b.ScheduleStatusID,2,6,3,7,5,1,4),a.BoardID DESC"
	result = execute_query_sql(sql)

	if result != ():
		for i in range(0,len(result)):
			temp = [result[i][0],result[i][1],result[i][2]]
			design_list.append(temp)

	design_status_list = get_order_status_list(list=design_status_list)

	return [design_status_list,design_list]

def get_all_designs_feedbacks():

	wwid=session.get('wwid')
	username = session.get('username')

	design_list = []
	design_status_list = []

	sql = "SELECT DISTINCT c.ScheduleTypeName FROM BoardDetails a, ScheduleTable b,ScheduleStatusType c WHERE a.BoardID=b.BoardID AND b.ScheduleStatusID=c.ScheduleID AND b.ScheduleStatusID <> 3 ORDER BY FIELD(b.ScheduleStatusID,2,6,3,7,5,1,4),a.BoardID DESC"
	result = execute_query_sql(sql)

	if result != ():
		for i in range(0,len(result)):
			design_status_list.append(result[i][0])


	sql = "SELECT a.BoardID,a.BoardName,c.ScheduleTypeName,b.ScheduleStatusID FROM BoardDetails a, ScheduleTable b,ScheduleStatusType c WHERE a.BoardID=b.BoardID AND b.ScheduleStatusID=c.ScheduleID ORDER BY FIELD(b.ScheduleStatusID,2,6,3,7,5,1,4),a.BoardID DESC"
	result = execute_query_sql(sql)

	if result != ():
		for i in range(0,len(result)):
			temp = [result[i][0],result[i][1],result[i][2]]
			design_list.append(temp)

	design_status_list = get_order_status_list(list=design_status_list)

	return [design_status_list,design_list]

def get_my_designs_feedbacks():

	wwid=session.get('wwid')
	#wwid = '10644414'
	username = session.get('username')

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	is_admin = False
	if has_admin_access == "yes":
		is_admin = True

	design_list = []
	design_status_list = []

	sql = "SELECT B.BoardID FROM BoardDetails B,ScheduleStatusType S1, ScheduleTable S2 WHERE S2.BoardID = B.BoardID AND S1.ScheduleID = S2.ScheduleStatusID ORDER BY FIELD(S2.ScheduleStatusID,2,6,3,7,5,1,4),B.BoardID DESC"
	bnames = execute_query_sql(sql)

	for j in bnames:

		sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
		val = (j[0],)
		sku_plat = execute_query(sql,val)

		present = False

		if is_admin:
			present = True

		if(present == False):
			sql = "SELECT * FROM BoardDetails WHERE BoardID = %s AND ((DesignLeadWWID = %s) OR (CADLeadWWID = %s) OR (PIFLeadWWID = %s))"
			val = (j[0],wwid,wwid,wwid)
			result = execute_query(sql,val)

			present = False
			if result != ():
				present = True	

		if(present == False):
			sql = "SELECT * FROM BoardDetails a LEFT JOIN HomeTable b ON a.DesignManagerWWID=b.UserName WHERE BoardID = %s AND b.WWID = %s"
			val = (j[0],wwid)
			result = execute_query(sql,val)

			if result != ():
				present = True	

		# for component review
		if(present == False):

			sql = "SELECT C3.CategoryLeadWWID,C2.PrimaryWWID,C2.SecondaryWWID,C3.CategoryLeadWWID1 FROM  ComponentReview C2,CategoryLeadTable C3,ComponentType C1 WHERE C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C1.ComponentID = C2.ComponentID AND C1.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID=%s"
			val = (sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
			primary_des = execute_query(sql,val)
			for i in primary_des:
				if(str(i[0]) == wwid or str(i[1]) == wwid or wwid in str(i[2]) or wwid in str(i[3])):
					present = True
					break

		# for component design
		if(present == False):

			sql = "SELECT C3.CategoryLeadWWID,C2.PrimaryWWID,C2.SecondaryWWID,C3.CategoryLeadWWID1 FROM  ComponentDesign C2,CategoryLeadTable C3,ComponentType C1 WHERE C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C1.ComponentID = C2.ComponentID AND C1.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID=%s"
			val = (sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
			primary_des = execute_query(sql,val)
			for i in primary_des:
				if(str(i[0]) == wwid or str(i[1]) == wwid or wwid in str(i[2]) or wwid in str(i[3])):
					present = True
					break

		if(present == True):
			sql = "SELECT B.BoardID,B.BoardName,S1.ScheduleTypeName,S1.ScheduleID FROM BoardDetails B, HomeTable H, DesignCalendar D2, ScheduleStatusType S1, ScheduleTable S2 WHERE B.DesignLeadWWID = H.WWID AND B.BoardID = D2.BoardID AND S2.BoardID = B.BoardID AND S1.ScheduleID = S2.ScheduleStatusID AND B.BoardID = %s ORDER BY S1.ScheduleTypeName  "
			val = (j[0],)
			blist = execute_query(sql,val)

			if blist != ():
				for i in range(0,len(blist)):
					design_list.append([blist[i][0],blist[i][1],blist[i][2]])

					if blist[i][2] not in design_status_list:
						if blist[i][3] not in [3,'3']:
							design_status_list.append(blist[i][2])

	design_status_list = get_order_status_list(list=design_status_list)

	return [design_status_list,design_list]

@app.route("/get_interfaces_ajax_feedbacks",methods = ['POST', 'GET'])
def get_interfaces_ajax_feedbacks():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]
	btn_value = data["btn"]

	result = {}

	sql = "SELECT * FROM BoardDetails WHERE BoardID = %s AND (DesignLeadWWID = %s OR CADLeadWWID = %s)"	
	val = (boardid,wwid,wwid)		
	rs_design_layout_lead = execute_query(sql,val)

	is_design_layout_lead = False
	if rs_design_layout_lead != ():
		is_design_layout_lead = True


	comp_status_list = []
	comp_list = []

	if btn_value == "All":
		temp_data = []
		temp_data = get_all_interfaces_feedbacks(boardid=boardid)

	else:
		temp_data = []
		# for design & Layout Lead, Managers - both My interface and All interface should be same and have edit access, as we dont have any mapping for design/layout lead and managers at backend properly
		if is_design_layout_lead:
			temp_data = get_all_interfaces_feedbacks(boardid=boardid)
		else:
			temp_data = get_my_interfaces_feedbacks(boardid=boardid)

	result["comp_list"] = json.dumps(temp_data[1])
	#result["comp_status_list"] = json.dumps(get_status_list_sorted(data_list=temp_data[0]))
	result["comp_status_list"] = json.dumps(get_order_status_list(list=temp_data[0]))

	return jsonify(result)


@app.route("/get_all_applicable_interface",methods = ['POST', 'GET'])
def get_all_applicable_interface():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]

	result = {}

	comp_status_list = []
	comp_list = []
	comp_selected_list = []

	temp_data = []
	temp_data = get_all_interfaces(boardid=boardid)

	result["comp_list"] = json.dumps(temp_data[1])
	#result["comp_status_list"] = json.dumps(get_status_list_sorted(data_list=temp_data[0]))
	result["comp_status_list"] = json.dumps(get_order_status_list(list=temp_data[0]))

	sql = "SELECT ComponentID FROM BoardReviewDesigner WHERE BoardID = %s AND IsPdgDesignSubmitted = %s"
	val = (boardid,"yes")
	comp_selected_list_rs = execute_query(sql,val)

	if comp_selected_list_rs != ():
		for i in range(0,len(comp_selected_list_rs)):
			comp_selected_list.append(comp_selected_list_rs[i][0])

	result["comp_selected_list"] = json.dumps(comp_selected_list)

	return jsonify(result)

def get_all_interfaces_feedbacks(boardid=0):

	wwid=session.get('wwid')
	username = session.get('username')

	comp_status_list = []
	comp_list = []

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	if sku_plat != ():
		sql = "SELECT DISTINCT S2.ScheduleTypeName FROM BoardReviewDesigner B1, ComponentType C1, ComponentReview C2, ScheduleTableComponent S1,ScheduleStatusType S2 WHERE B1.ComponentID = C1.ComponentID AND B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C2.ComponentID AND C2.IsValid = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID = %s AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S2.ScheduleID = S1.ScheduleStatusID ORDER BY FIELD(S1.ScheduleStatusID,2,6,3,7,5,1,4), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
		val = (boardid,"yes",True,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		result = execute_query(sql,val)

		if result != ():
			for i in range(0,len(result)):
				comp_status_list.append(result[i][0])

		sql = "SELECT DISTINCT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1, ComponentType C1, ComponentReview C2, ScheduleTableComponent S1,ScheduleStatusType S2 WHERE B1.ComponentID = C1.ComponentID AND B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C2.ComponentID AND C2.IsValid = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID = %s AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S2.ScheduleID = S1.ScheduleStatusID ORDER BY FIELD(S1.ScheduleStatusID,2,6,3,7,5,1,4), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
		val = (boardid,"yes",True,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		complist = execute_query(sql,val)

		if complist != ():
			for i in range(0,len(complist)):
				temp = [complist[i][0],complist[i][1],complist[i][2],complist[i][3]]
				comp_list.append(temp)

	comp_status_list = get_order_status_list(list=comp_status_list)

	return [comp_status_list,comp_list]

def get_my_interfaces_feedbacks(boardid=0):

	wwid=session.get('wwid')
	username = session.get('username')

	comp_list = []
	comp_status_list = []

	like_wwid = '%' + str(wwid) + '%'

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	if sku_plat != ():

		sql = "SELECT DISTINCT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1, ComponentType C1, ComponentReview C2, ScheduleTableComponent S1,ScheduleStatusType S2 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND C2.IsValid = %s AND B1.ComponentID = C2.ComponentID AND B1.BoardID = S1.BoardID AND B1.ComponentID = S1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID =%s AND (C2.PrimaryWWID = %s OR C2.SecondaryWWID LIKE %s) "

		sql += " UNION "
		sql += "SELECT DISTINCT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1, ComponentType C1, ComponentDesign C2, ScheduleTableComponent S1,ScheduleStatusType S2 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND C2.IsValid = %s AND B1.ComponentID = C2.ComponentID AND B1.BoardID = S1.BoardID AND B1.ComponentID = S1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID =%s AND (C2.PrimaryWWID = %s OR C2.SecondaryWWID LIKE %s) "

		sql += " UNION "

		sql += "SELECT DISTINCT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1, ComponentType C1, CategoryLeadTable C2, ScheduleTableComponent S1,ScheduleStatusType S2 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND C1.CategoryID = C2.CategoryID AND B1.BoardID = S1.BoardID AND B1.ComponentID = S1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID =%s AND (C2.CategoryLeadWWID = %s OR C2.CategoryLeadWWID1 LIKE %s) "
		val = (boardid,"yes",True,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],wwid,like_wwid,boardid,"yes",True,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],wwid,like_wwid,boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],wwid,like_wwid)

		complist = execute_query(sql,val)

		if complist != ():
			for i in range(0,len(complist)):
				temp = [complist[i][0],complist[i][1],complist[i][2],complist[i][3]]
				comp_list.append(temp)
				comp_status_list.append(complist[i][2])

		comp_status_list = list(set(comp_status_list))

	comp_status_list = get_order_status_list(list=comp_status_list)

	return [comp_status_list,comp_list]

def get_feedbacks_data_page(data,boardid,compid,complist,sku_plat,board_status,my_designs_id,my_components_id):

	is_admin = session.get('is_admin')
	wwid=session.get('wwid')
	username = session.get('username')

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	data[compid] = {}
	data[compid]["add_new_feedbacks_btn_enable"] = False
	data[compid]['child_issues_parent_comment_id'] = []
	feedbacks = []
	saved_feedbacks = []

	data[compid]["comp_name"] = ''
	data[compid]["comp_status_id"] = 0
	data[compid]["comp_status"] = ''
	data[compid]["is_reopen_enabled"] = False
	data[compid]["max_feedback_number"] = 0
	data[compid]["max_feedback_count"] = 0
	data[compid]["enable_btn_for_signoff"] = False

	data[compid]['design_document_name_list'] = []
	is_rev0p6_design = False
	is_rev1p0_design = True

	data[compid]['area_of_quality_issue'] = []
	data[compid]['quality_issue'] = ""
	data[compid]['quality_issue_comments'] = ""
	data[compid]['quality_issue_comments_summary'] = ""
	data[compid]['quality_not_met_counter'] = 0
	data[compid]['show_feedback_table'] = False
	data[compid]['is_basic_quality_check_submitted'] = False
	data[compid]['freeze_notmet_basic_quality_check'] = False

	sql = "SELECT a.BoardID,a.ComponentID,a.CommentID,a.QualityCheck,a.AreaOfQualityIssue,a.Comments,a.QualityNotMetCounter,a.IsSubmitted,a.UpdatedBy,a.UpdatedOn,b.AreaofIssue,IFNULL(c.UserName,'') FROM BasicDesignQualityCheck a LEFT JOIN AreaOfQualityIssue b ON a.AreaOfQualityIssue = b.ID LEFT JOIN HomeTable c ON a.UpdatedBy = c.WWID WHERE a.BoardID = %s AND a.ComponentID = %s ORDER BY a.CommentID DESC"
	val = (boardid,compid)
	basic_quality_check_rs = execute_query(sql,val)

	for temp_row in basic_quality_check_rs:

		if temp_row[7] != "yes":
			data[compid]['is_basic_quality_check_submitted'] = False
			data[compid]['show_feedback_table'] = False
			break

	for temp_row in basic_quality_check_rs:

		if temp_row[3] == "notmet":
			data[compid]['quality_issue_comments_summary'] += '<span style="color:red;">Not met </span>- '+str(temp_row[10])+'<br>'+str(temp_row[5])+'<br>'+'<span style="color:lightGray;">Updated By:&nbsp;&nbsp;'+str(temp_row[11])+'</span><br><br>'
			data[compid]['quality_not_met_counter'] += 1

	if basic_quality_check_rs != ():
		data[compid]['quality_issue'] = basic_quality_check_rs[0][10]
		data[compid]['quality_issue_comments'] = basic_quality_check_rs[0][5]
		#data[compid]['quality_not_met_counter'] = basic_quality_check_rs[0][6]

		if (basic_quality_check_rs[0][3] == "met") and (basic_quality_check_rs[0][7] == "yes"):
			data[compid]['show_feedback_table'] = True
		else:
			data[compid]['show_feedback_table'] = False

		if (basic_quality_check_rs[0][3] == "notmet") and (basic_quality_check_rs[0][7] == "yes"):
			data[compid]['freeze_notmet_basic_quality_check'] = True

		if basic_quality_check_rs[0][7] == "yes":
			data[compid]['is_basic_quality_check_submitted'] = True
		else:
			data[compid]['is_basic_quality_check_submitted'] = False

	sql = "SELECT ID,AreaofIssue FROM AreaOfQualityIssue ORDER BY AreaofIssue"
	area_of_quality_issue_rs = execute_query_sql(sql)

	for row in area_of_quality_issue_rs:
		data[compid]['area_of_quality_issue'].append([row[0],row[1]])

	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	timeline_rs = execute_query(sql,val)

	is_rev0p6_design = False
	is_rev1p0_design = True
	if timeline_rs != ():
		if timeline_rs[0][0] in [1,'1']:
			is_rev0p6_design = True
			is_rev1p0_design = False

	# to get complete user list
	sql = "SELECT DISTINCT WWID,UserName,RoleID,AdminAccess,EmailID FROM HomeTable ORDER BY UserName"
	user_list_rs = execute_query_sql(sql)

	user_list = []
	for row in user_list_rs:
		user_list.append([row[0],row[1],row[2],row[3],row[4]])

	#if complist != ():
	if complist != []:
		for i in range(0,len(complist)):
			if int(complist[i][0]) == int(compid):
				data[compid]["comp_name"] = complist[i][1]
				data[compid]["comp_status_id"] = complist[i][3]
				data[compid]["comp_status"] = complist[i][2]

				# basic design quality check 
				if complist[i][2] == "Signed-Off":
					data[compid]['show_feedback_table'] = True
					data[compid]['is_basic_quality_check_submitted'] = True

				# for rev0p6 design, change singed-off to closed
				if is_rev0p6_design:
					if complist[i][2] == "Signed-Off":
						data[compid]["comp_status"] = "Closed"				

				if data[compid]["comp_status_id"] in [2,3,6]:

					if board_status in [2,6]:

						if is_admin:
							data[compid]["add_new_feedbacks_btn_enable"] = True

						if (int(boardid) in my_designs_id) and (int(compid) in my_components_id):
							data[compid]["add_new_feedbacks_btn_enable"] = True

				if board_status in [2,3,6]:
					if data[compid]["comp_status_id"] in [1,5,7]:
						if (int(boardid) in my_designs_id) and (int(compid) in my_components_id):
							data[compid]["is_reopen_enabled"] = True

						if is_admin:
							data[compid]["is_reopen_enabled"] = True


	# to keep view mode for these 2 design
	if boardid in [60,61,'60','61']:
		data[compid]["add_new_feedbacks_btn_enable"] = False
		data[compid]["is_reopen_enabled"] = False

	#sql = "SELECT count(*) FROM BoardReview WHERE HasChild IS NULL AND BoardID = %s AND ComponentID = %s"
	sql = "SELECT count(*) FROM BoardReview WHERE HasChild IS NULL AND BoardID = %s AND ComponentID = %s"
	val = (boardid,compid)
	result_temp = execute_query(sql,val)

	if result_temp != ():
		data[compid]["max_feedback_count"] = copy.deepcopy(result_temp[0][0])

	sql = "SELECT IFNULL(MAX(FeedbackNo),0) FROM BoardReview WHERE BoardID = %s AND ComponentID = %s"
	val = (boardid,compid)
	result = execute_query(sql,val)

	if result != ():
		data[compid]["max_feedback_number"] = copy.deepcopy(result[0][0])

	if data[compid]["max_feedback_number"] > data[compid]["max_feedback_count"]:
		data[compid]["max_feedback_count"] = copy.deepcopy(data[compid]["max_feedback_number"])
	else:
		data[compid]["max_feedback_number"] = copy.deepcopy(data[compid]["max_feedback_count"])

	sql = "SELECT * FROM BoardReview WHERE HasChild IS NULL AND BoardID = %s AND ComponentID = %s AND Submitted_Designer = %s AND (Submitted_Reviewer2 IS NULL OR Submitted_Reviewer2 = %s)"
	val = (boardid,compid,"yes","no")
	result_signoff_temp = execute_query(sql,val)

	if result_signoff_temp != ():
		data[compid]["enable_btn_for_signoff"] = True

	# Graded by Design Team As PDG data
	sql = "SELECT DISTINCT IFNULL(B1.PDG,''),IFNULL(B1.CommentDesigner,''),IFNULL(B1.PDG_Electrical,''),IFNULL(B1.CommentElectrical,''),IsPdgElectricalSubmitted FROM BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.ComponentID = %s" 
	val = (boardid,compid)
	result = execute_query(sql,val)

	data[compid]["comp_pdg"] = ''
	data[compid]["pdg_comment"] = ''
	data[compid]["comp_pdg_elec"] = ''
	data[compid]["pdg_comment_elec"] = ''
	data[compid]["is_pdg_elect_submitted"] = False

	if result != ():
		data[compid]["comp_pdg"] = result[0][0]
		data[compid]["pdg_comment"] = result[0][1]

		if result[0][2] == 'NULL':
			data[compid]["comp_pdg_elec"] = ''
		else:
			data[compid]["comp_pdg_elec"] = result[0][2]

		data[compid]["pdg_comment_elec"] = result[0][3]

		if result[0][4] == "yes":
			data[compid]["is_pdg_elect_submitted"] = True

	data[compid]["pdg_not_met_checked"] = ""
	data[compid]["pdg_met_checked"] = ""

	if data[compid]["comp_pdg_elec"] == "Met":
		data[compid]["pdg_met_checked"] = "checked"
		data[compid]["pdg_not_met_checked"] = ""

	elif data[compid]["comp_pdg_elec"] == "Not Met":
		data[compid]["pdg_not_met_checked"] = "checked"
		data[compid]["pdg_met_checked"] = ""

	# Electrical, Design Owner & Status
	data[compid]["comp_electrical_owner"] = ''
	data[compid]["comp_design_owner"] = ''
	#data[compid]["comp_status"] = ''

	# to check file presence
	data[compid]["FileName"] = ''
	data[compid]["BoardFileName"] = ''
	data[compid]["SchematicsName"] = ''
	data[compid]["StackupFileName"] = ''
	data[compid]["LengthReportFileName"] = ''
	data[compid]["OthersFileName"] = ''

	sql = "SELECT IFNULL(FileName,''),IFNULL(BoardFileName,''),IFNULL(SchematicsName,''),IFNULL(StackupFileName,''),IFNULL(LengthReportFileName,''),IFNULL(OthersFileName,'') FROM UploadSignOffFiles WHERE BoardID = %s AND ComponentID = %s" 
	val = (boardid,compid)
	result = execute_query(sql,val)

	design_document_name_list = []

	if result != ():
		data[compid]["FileName"] = result[0][0]
		data[compid]["BoardFileName"] = result[0][1]
		data[compid]["SchematicsName"] = result[0][2]
		data[compid]["StackupFileName"] = result[0][3]
		data[compid]["LengthReportFileName"] = result[0][4]
		data[compid]["OthersFileName"] = result[0][5]

		design_document_name_list.append([result[0][1],"Board File"])
		design_document_name_list.append([result[0][2],"Schematics File"])
		design_document_name_list.append([result[0][3],"Stackup File"])
		design_document_name_list.append([result[0][4],"Lenght Report File"])
		design_document_name_list.append([result[0][5],"Others File"])

		design_document_name_list = [row for row in design_document_name_list if row[0] != '']

		data[compid]['design_document_name_list'] = copy.deepcopy(design_document_name_list)

	if sku_plat != ():
		# component primary owners and status
		sql = "SELECT H.UserName FROM HomeTable H, ComponentReview C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H.WWID ORDER BY C2.ComponentID"
		val = (compid,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		result = execute_query(sql,val)

		if result != ():
			data[compid]["comp_electrical_owner"] = result[0][0]

		sql = "SELECT H.UserName FROM HomeTable H, ComponentDesign C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H.WWID ORDER BY C2.ComponentID"
		val = (compid,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		result = execute_query(sql,val)

		if result != ():
			data[compid]["comp_design_owner"] = result[0][0]

	# list of edit discard feedbacks at board level
	sql = "SELECT BoardID,ComponentID,CommentID,WWIDreviewer,WWIDdesigner FROM BoardReviewTemp WHERE BoardID = %s"
	val = (boardid,)
	edit_discard_rs = execute_query(sql,val)

	# feedbacks data
	sql = "(SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,IFNULL(B1.Comment_Reviewer,''),B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time,B2.PDG,B2.CommentDesigner,'',S.ScheduleStatusID,B1.IsImported,B1.ImportedFromDesignID,B1.AddedByDesignTeam,B1.RiskLevelSubmitted,B1.FeedbackNo,B1.is_edit_save_flag,B1.is_edit_save_flag_design FROM BoardReview B1 INNER JOIN BoardReviewDesigner B2 ON B1.BoardID = B2.BoardID AND B1.ComponentID = B2.ComponentID INNER JOIN ScheduleTable S ON B1.BoardID = S.BoardID INNER JOIN DesignCalendar D5 ON B1.BoardID = D5.BoardID WHERE B1.ParentCommentID = 0 AND B1.Submitted = 'yes' AND AddedByDesignTeam = 'yes' AND B1.BoardID = %s AND B1.ComponentID = %s ORDER BY CommentID)" 
	sql += " UNION ALL "
	sql += "(SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,IFNULL(B1.Comment_Reviewer,''),B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time,B2.PDG,B2.CommentDesigner,'',S.ScheduleStatusID,B1.IsImported,B1.ImportedFromDesignID,B1.AddedByDesignTeam,B1.RiskLevelSubmitted,B1.FeedbackNo,B1.is_edit_save_flag,B1.is_edit_save_flag_design FROM BoardReview B1 INNER JOIN BoardReviewDesigner B2 ON B1.BoardID = B2.BoardID AND B1.ComponentID = B2.ComponentID INNER JOIN ScheduleTable S ON B1.BoardID = S.BoardID INNER JOIN DesignCalendar D5 ON B1.BoardID = D5.BoardID WHERE B1.ParentCommentID = 0 AND B1.Submitted = 'yes' AND AddedByDesignTeam IS NULL AND B1.BoardID = %s AND B1.ComponentID = %s ORDER BY CommentID)" 
	val = (boardid,compid,boardid,compid)
	parents = execute_query(sql,val)

	sql = "SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,IFNULL(B1.Comment_Reviewer,''),B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time,B2.PDG,B2.CommentDesigner,'',S.ScheduleStatusID,B1.IsImported,B1.ImportedFromDesignID,B1.AddedByDesignTeam,B1.RiskLevelSubmitted,B1.FeedbackNo,B1.is_edit_save_flag,B1.is_edit_save_flag_design FROM BoardReview B1 INNER JOIN BoardReviewDesigner B2 ON B1.BoardID = B2.BoardID AND B1.ComponentID = B2.ComponentID INNER JOIN ScheduleTable S ON B1.BoardID = S.BoardID INNER JOIN DesignCalendar D5 ON B1.BoardID = D5.BoardID WHERE B1.ParentCommentID <> 0 AND B1.Submitted = 'yes' AND B1.BoardID = %s AND B1.ComponentID = %s ORDER BY CommentID"
	val = (boardid,compid)
	children = execute_query(sql,val)

	result = []

	for i in parents:
		k=0
		result.append(i)
		parentid = i[2]
		while k<len(children):
			#if children[k][12] == parentid:
			if children[k][28] == str(parentid):
				result.append(children[k])
				#break
			k+=1

	f_no = 0


	for i in range(0,len(result)):

		f_row_span = 1
		f_no_td_enabled = True

		if result[i][12] == 0:
			f_no += 1							

		temp = []
		temp.append(result[i][0])
		temp.append(result[i][1])
		temp.append(result[i][2])

		temp.append(f_no_td_enabled)
		temp.append(f_row_span)
		temp.append(f_no)

		temp.append(result[i][24])
		temp.append(result[i][3])

		if result[i][4] is not None:
			temp.append(result[i][4].replace("\n","<br>"))
		else:
			temp.append(result[i][4])

		temp.append(result[i][5])

		if result[i][6] != None:
			temp1 = result[i][6].replace("---------","<br>")
			temp2 = temp1.replace("--",'<br><br><span style="color:grey; font-size: smaller;">Updated By: ')
			temp3 = temp2.replace(", Date: ",'<br>Updated On: ')
			if (temp3.find('Updated By:') != -1):
				temp3 += '</span>'
			temp.append(temp3.replace("\n","<br>"))
		else:
			temp.append('')

		temp.append(result[i][7])
		temp.append(result[i][23].replace("\n","<br>"))

		#download attachement
		if result[i][26] not in ('No File',None,''):
			temp.append(True)
		else:
			temp.append(False)

		if result[i][8] != None:
			temp.append(result[i][8])
		else:
			temp.append('')

		if result[i][9] != None:
			temp1 = result[i][9].replace("---------","<br>")
			temp2 = temp1.replace("--",'<br><span style="color:grey; font-size: smaller;">Updated By: ')
			temp3 = temp2.replace(", Date: ",'<br>Updated On: ')
			if (temp3.find('Updated By:') != -1):
				temp3 += '</span>'
			temp.append(temp3.replace("\n","<br>"))
		else:
			temp.append('')				

		if result[i][27] not in ('No File',None,''):
			temp.append(True)
		else:
			temp.append(False)

		if result[i][19] != None:
			temp.append(result[i][19])
		else:
			temp.append('')

		if result[i][20] != None:
			temp1 = result[i][20].replace("---------","<br>")
			temp2 = temp1.replace("--",'<br><span style="color:grey; font-size: smaller;">Updated By: ')
			temp3 = temp2.replace(", Date: ",'<br>Updated On: ')
			if (temp3.find('Updated By:') != -1):
				temp3 += '</span>'
			temp.append(temp3.replace("\n","<br>"))
		else:
			temp.append('')

		if result[i][21] != None:
			temp.append(result[i][21])
		else:
			temp.append('')

		#temp.append('') #bgcolor

		# bgcolor for added by design team
		if result[i][38] == "yes":	#20
			temp.append('')
		else:
			temp.append('')

		temp.append(result[i][39])	#risk level submitted

		temp.append(result[i][15])	# designer saved
		temp.append(result[i][16])	# designer submitted
		temp.append(result[i][14])	# electrical submitted 1st part
		temp.append(result[i][18])	# subitted reviewer2

		temp.append(result[i][36]) # is imported

		temp.append(result[i][37]) # imported from design ID

		if result[i][36] == "yes":	# for non editable imported feedbacks
			#temp.append('readonly')
			temp.append('')
		else:
			temp.append('')

		temp.append(False)	# 29 - for issues highlighted by design / electrical team
		temp.append(1)	# 30 - for issues highlighted by design / electrical team - rowspan
		temp.append(result[i][38]) #added by design team

		# actual parent id #32
		if result[i][28] is None:
			temp.append(0)
		else:
			temp.append(result[i][28])

		# paren comment id #33
		if result[i][12] is not None:
			temp.append(result[i][12])
		else:
			temp.append(0)

		# is valid child issue saved #34
		temp.append(False)

		# edit values for child saved issues - dummy - 5 fields
		temp.append("")
		temp.append("")
		temp.append("")
		temp.append("")
		temp.append(False)
		temp.append(0)

		# tag for child issue last part - #41 & 42
		temp.append(result[i][18])
		temp.append(result[i][16])

		temp.append(result[i][2])	# 43 - for edit comment id, to handle child feedback as well 

		# for fixed feedback Number  #44
		if result[i][40] == 0:
			temp.append(f_no)
		else:
			temp.append(result[i][40])

		if result[i][26] not in ('No File',None,''):	#45
			temp.append("Download File\nFile Name: "+result[i][26])
		else:
			temp.append("")

		if result[i][27] not in ('No File',None,''):	#46
			temp.append("Download File\nFile Name: "+result[i][27])
		else:
			temp.append("")

		# 47 - edit save flag for design team to display 'Feedback moved to edit mode by <username>'
		if str(result[i][30]) == str(wwid):
			temp.append(False)
		else:
			if str(result[i][42]) == str(1):
				temp.append(True)
			else:
				temp.append(False)

		# 48 - display text for above (47)
		edit_save_design_display_text = ''
		for row in user_list_rs:
			if str(result[i][30]) == str(row[0]):
				edit_save_design_display_text = "Feedback is moved to edit mode by " + str(row[1])
				break

		temp.append(edit_save_design_display_text)

		# 49, 50 - edit discard flag
		temp.append("block")
		temp.append("none")

		feedbacks.append(temp)

	# for rowspan of child feedbacks
	for i in range(0,len(feedbacks)):

		f_row_span = 0
		feedbacks[i][3] = True

		for j in range(i,len(feedbacks)):
			if str(feedbacks[i][5]) == str(feedbacks[j][5]):
				f_row_span += 1

		feedbacks[i][4] = copy.deepcopy(f_row_span)

		if i>0:
			if str(feedbacks[i][5]) == str(feedbacks[i-1][5]):
				feedbacks[i][3] = False
				feedbacks[i][20] = 'bgcolor="#effbff"'
				feedbacks[i-1][20] = 'bgcolor="#effbff"'

	data[compid]["feedbacks"] = copy.deepcopy(feedbacks)

	# for child issue saved data, putting into main feedback row not in saved feedbacks
	for i in range(0,len(data[compid]["feedbacks"])):

		for j in range(0,len(children)):
				
			# for cild issue last part to be filled by electrical owner
			if str(data[compid]["feedbacks"][i][2]) == str(children[j][28]):

				if children[j][19] != None:
					data[compid]["feedbacks"][i][17] = copy.deepcopy(children[j][19])
				else:
					data[compid]["feedbacks"][i][17] = ''

				if children[j][20] != None:
					temp1 = children[j][20].replace("---------","<br>")
					temp2 = temp1.replace("--",'<br><span style="color:grey; font-size: smaller;">Updated By: ')
					temp3 = temp2.replace(", Date: ",'<br>Updated On: ')
					if (temp3.find('Updated By:') != -1):
						temp3 += '</span>'
					data[compid]["feedbacks"][i][18] = copy.deepcopy(temp3)
				else:
					data[compid]["feedbacks"][i][18] = ''

				if children[j][21] != None:
					data[compid]["feedbacks"][i][19] = copy.deepcopy(children[j][21])
				else:
					data[compid]["feedbacks"][i][19] = ''

				data[compid]["feedbacks"][i][41] = copy.deepcopy(children[j][18])
				data[compid]["feedbacks"][i][42] = copy.deepcopy(children[j][16])

				data[compid]["feedbacks"][i][43] = children[j][2]

	# for update flag of edit discard icon in UI
	for i in range(0,len(data[compid]["feedbacks"])):
		for j in range(0,len(edit_discard_rs)):
			if (edit_discard_rs[j][1] == data[compid]["feedbacks"][i][1]) and (edit_discard_rs[j][2] == data[compid]["feedbacks"][i][43]):
				if is_admin or is_elec_owner:
					if str(wwid) == str(edit_discard_rs[j][3]):
						data[compid]["feedbacks"][i][49] = "none"
						data[compid]["feedbacks"][i][50] = "block"

				if is_design_owner or is_layout_owner:
					if str(wwid) == str(edit_discard_rs[j][4]):
						data[compid]["feedbacks"][i][49] = "none"
						data[compid]["feedbacks"][i][50] = "block"

	# saved edited feedbacks for highlighting
	sql = "(SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,IFNULL(B1.Comment_Reviewer,''),B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time,B2.PDG,B2.CommentDesigner,'',S.ScheduleStatusID,B1.IsImported,B1.ImportedFromDesignID,B1.AddedByDesignTeam,B1.RiskLevelSubmitted,B1.FeedbackNo,B1.is_edit_save_flag FROM BoardReview B1 INNER JOIN BoardReviewDesigner B2 ON B1.BoardID = B2.BoardID AND B1.ComponentID = B2.ComponentID INNER JOIN ScheduleTable S ON B1.BoardID = S.BoardID INNER JOIN DesignCalendar D5 ON B1.BoardID = D5.BoardID WHERE B1.ParentCommentID = 0 AND B1.Saved = 'yes' AND (B1.Submitted IS NULL OR B1.Submitted = 'no') AND AddedByDesignTeam = 'yes' AND B1.is_edit_save_flag = %s AND B1.BoardID = %s AND B1.ComponentID = %s AND B1.WWIDreviewer <> %s ORDER BY CommentID)" 
	sql += " UNION ALL "
	sql += "(SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,IFNULL(B1.Comment_Reviewer,''),B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time,B2.PDG,B2.CommentDesigner,'',S.ScheduleStatusID,B1.IsImported,B1.ImportedFromDesignID,B1.AddedByDesignTeam,B1.RiskLevelSubmitted,B1.FeedbackNo,B1.is_edit_save_flag FROM BoardReview B1 INNER JOIN BoardReviewDesigner B2 ON B1.BoardID = B2.BoardID AND B1.ComponentID = B2.ComponentID INNER JOIN ScheduleTable S ON B1.BoardID = S.BoardID INNER JOIN DesignCalendar D5 ON B1.BoardID = D5.BoardID WHERE B1.ParentCommentID = 0 AND B1.Saved = 'yes' AND (B1.Submitted IS NULL OR B1.Submitted = 'no') AND AddedByDesignTeam IS NULL AND B1.is_edit_save_flag = %s AND B1.BoardID = %s AND B1.ComponentID = %s AND B1.WWIDreviewer <> %s ORDER BY CommentID)" 
	val = (1,boardid,compid,wwid,1,boardid,compid,wwid)
	parents_edited_saved = execute_query(sql,val)

	temp_edited_feedbacks = []
	if parents_edited_saved != ():
		for row in parents_edited_saved:
			temp_row = []
			temp_row.append(row[40])

			user_name_temp = ''

			for user_row in user_list:
				if row[29] == user_row[0]:
					user_name_temp = copy.deepcopy(user_row[1])
					break

			temp_row.append(user_name_temp)

			temp_edited_feedbacks.append(temp_row)

	data[compid]["edited_saved_feedbacks"] = copy.deepcopy(temp_edited_feedbacks)

	# saved feedbacks
	sql = "(SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,IFNULL(B1.Comment_Reviewer,''),B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time,B2.PDG,B2.CommentDesigner,'',S.ScheduleStatusID,B1.IsImported,B1.ImportedFromDesignID,B1.AddedByDesignTeam,B1.RiskLevelSubmitted,B1.FeedbackNo,B1.is_edit_save_flag FROM BoardReview B1 INNER JOIN BoardReviewDesigner B2 ON B1.BoardID = B2.BoardID AND B1.ComponentID = B2.ComponentID INNER JOIN ScheduleTable S ON B1.BoardID = S.BoardID INNER JOIN DesignCalendar D5 ON B1.BoardID = D5.BoardID WHERE B1.ParentCommentID = 0 AND B1.Saved = 'yes' AND (B1.Submitted IS NULL OR B1.Submitted = 'no') AND AddedByDesignTeam = 'yes' AND B1.BoardID = %s AND B1.ComponentID = %s AND (B1.WWIDreviewer = %s OR B1.WWIDdesigner = %s) ORDER BY CommentID)" 
	sql += " UNION ALL "
	sql += "(SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,IFNULL(B1.Comment_Reviewer,''),B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time,B2.PDG,B2.CommentDesigner,'',S.ScheduleStatusID,B1.IsImported,B1.ImportedFromDesignID,B1.AddedByDesignTeam,B1.RiskLevelSubmitted,B1.FeedbackNo,B1.is_edit_save_flag FROM BoardReview B1 INNER JOIN BoardReviewDesigner B2 ON B1.BoardID = B2.BoardID AND B1.ComponentID = B2.ComponentID INNER JOIN ScheduleTable S ON B1.BoardID = S.BoardID INNER JOIN DesignCalendar D5 ON B1.BoardID = D5.BoardID WHERE B1.ParentCommentID = 0 AND B1.Saved = 'yes' AND (B1.Submitted IS NULL OR B1.Submitted = 'no') AND AddedByDesignTeam IS NULL AND B1.BoardID = %s AND B1.ComponentID = %s AND (B1.WWIDreviewer = %s OR B1.WWIDdesigner = %s) ORDER BY CommentID)" 
	val = (boardid,compid,wwid,wwid,boardid,compid,wwid,wwid)
	parents = execute_query(sql,val)

	sql = "SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,IFNULL(B1.Comment_Reviewer,''),B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time,B2.PDG,B2.CommentDesigner,'',S.ScheduleStatusID,B1.IsImported,B1.ImportedFromDesignID,B1.AddedByDesignTeam,B1.RiskLevelSubmitted,B1.FeedbackNo,B1.is_edit_save_flag FROM BoardReview B1 INNER JOIN BoardReviewDesigner B2 ON B1.BoardID = B2.BoardID AND B1.ComponentID = B2.ComponentID INNER JOIN ScheduleTable S ON B1.BoardID = S.BoardID INNER JOIN DesignCalendar D5 ON B1.BoardID = D5.BoardID WHERE B1.ParentCommentID <> 0 AND B1.Saved = 'yes' AND (B1.Submitted IS NULL OR B1.Submitted = 'no') AND B1.BoardID = %s AND B1.ComponentID = %s AND (B1.WWIDreviewer = %s OR B1.WWIDdesigner = %s) ORDER BY CommentID" 
	val = (boardid,compid,wwid,wwid)
	children = execute_query(sql,val)

	result = []

	for i in parents:
		k=0
		result.append(i)
		parentid = i[2]
		while k<len(children):
			#if children[k][12] == parentid:
			if children[k][28] == str(parentid):
				result.append(children[k])
				break
			k+=1

	for i in range(0,len(result)):

		f_row_span = 1
		f_no_td_enabled = True

		if result[i][12] == 0:
			f_no += 1							

		temp = []
		temp.append(result[i][0])
		temp.append(result[i][1])
		temp.append(result[i][2])

		temp.append(f_no_td_enabled)
		temp.append(f_row_span)
		temp.append(f_no)

		temp.append(result[i][24])
		temp.append(result[i][3])
		temp.append(result[i][4])
		temp.append(result[i][5])

		temp.append(result[i][6])

		temp.append(result[i][7])
		temp.append(result[i][23])

		#download attachement
		if result[i][26] == '':
			temp.append('No File')
		else:
			temp.append(result[i][26])

		temp.append(result[i][8])

		temp.append(result[i][9])

		temp.append(result[i][27])

		temp.append(result[i][19])

		temp.append(result[i][20])

		temp.append(result[i][21])

		#temp.append('') #bgcolor
		# bgcolor for added by design team
		if result[i][38] == "yes":	#20
			temp.append('')
		else:
			temp.append('')

		temp.append(result[i][39])	#risk level submitted
		temp.append(result[i][15])	# designer saved
		temp.append(result[i][16])	# designer submitted
		temp.append(result[i][14])	# electrical submitted 1st part
		temp.append(result[i][18])	# subitted reviewer2

		temp.append(result[i][36]) # is imported

		temp.append(result[i][37]) # imported from design ID

		if result[i][36] == "yes":	# for non editable imported feedbacks
			#temp.append('readonly')
			temp.append('')
		else:
			temp.append('')

		temp.append(False)	# for issues highlighted by design / electrical team
		temp.append(1)	# for issues highlighted by design / electrical team - rowspan
		temp.append(result[i][38]) #added by design team

		# actual parent id #32
		if result[i][28] is None:
			temp.append(0)
		else:
			temp.append(result[i][28])

		# paren comment id #33
		if result[i][12] is not None:
			temp.append(result[i][12])
		else:
			temp.append(0)

		# is valid child issue saved #34
		temp.append(False)

		# for fixed feedback number #35
		if result[i][40] == 0:
			temp.append(f_no)
		else:
			temp.append(result[i][40])

		temp.append(result[i][29])	# wwid for 1st part #36
		temp.append(result[i][41])	# is_edit_save_flag #37

		if (data[compid]["comp_status_id"] in [3,'3']) and (result[i][38] is not None):
			pass
		else:
			saved_feedbacks.append(temp)

	for i in range(0,len(saved_feedbacks)):

		f_row_span = 0
		saved_feedbacks[i][3] = True

		for j in range(i,len(saved_feedbacks)):
			if str(saved_feedbacks[i][5]) == str(saved_feedbacks[j][5]):
				f_row_span += 1

		saved_feedbacks[i][4] = copy.deepcopy(f_row_span)

		if i>0:
			if str(saved_feedbacks[i][5]) == str(saved_feedbacks[i-1][5]):
				saved_feedbacks[i][3] = False
				saved_feedbacks[i][20] = 'bgcolor="#effbff"'
				saved_feedbacks[i-1][20] = 'bgcolor="#effbff"'


	# for child issue saved data, putting into main feedback row not in saved feedbacks
	for i in range(0,len(data[compid]["feedbacks"])):

		for j in range(0,len(children)):

			if str(data[compid]["feedbacks"][i][2]) == str(children[j][12]):
				data[compid]["feedbacks"][i][34] = True

				data[compid]["feedbacks"][i][35] = str(children[j][24])
				data[compid]["feedbacks"][i][36] = str(children[j][6])
				data[compid]["feedbacks"][i][37] = str(children[j][7])
				data[compid]["feedbacks"][i][38] = str(children[j][23])

				if children[j][26] not in ['No File',None,'']:
					data[compid]["feedbacks"][i][39] = True

				data[compid]["feedbacks"][i][40] = children[j][2]
				
			# for cild issue last part to be filled by electrical owner
			if str(data[compid]["feedbacks"][i][2]) == str(children[j][28]):
				
				if children[j][19] != None:
					data[compid]["feedbacks"][i][17] = copy.deepcopy(children[j][19])
				else:
					data[compid]["feedbacks"][i][17] = ''

				if children[j][20] != None:
					temp1 = children[j][20].replace("---------","<br>")
					temp2 = temp1.replace("--",'<br><span style="color:grey; font-size: smaller;">Updated By: ')
					temp3 = temp2.replace(", Date: ",'<br>Updated On: ')
					if (temp3.find('Updated By:') != -1):
						temp3 += '</span>'
					data[compid]["feedbacks"][i][18] = copy.deepcopy(temp3)
				else:
					data[compid]["feedbacks"][i][18] = ''

				if children[j][21] != None:
					data[compid]["feedbacks"][i][19] = copy.deepcopy(children[j][21])
				else:
					data[compid]["feedbacks"][i][19] = ''

				data[compid]["feedbacks"][i][41] = copy.deepcopy(children[j][18])
				data[compid]["feedbacks"][i][42] = copy.deepcopy(children[j][16])

	# for table row alternate colors
	for i in range(0,len(data[compid]["feedbacks"])):
		if data[compid]["feedbacks"][i][20] == '':
			if i%2 == 0:
				data[compid]["feedbacks"][i][20] = 'bgcolor="#F7F8F9"'
			else:
				data[compid]["feedbacks"][i][20] = 'bgcolor="#ffffff"'

	# convert python to javascript json format
	data[compid]["saved_feedbacks"] = json.dumps(copy.deepcopy(saved_feedbacks))

	return data

@app.route("/get_feedbacks_import_issues",methods = ['POST', 'GET'])
def get_feedbacks_import_issues():
	
	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))

	board_id = copy.deepcopy(data['board_id'])
	comp_id = copy.deepcopy(data['comp_id'])
	import_boardid = copy.deepcopy(data['import_boardid'])
	import_issue_status = copy.deepcopy(data['import_issue_status'])

	# saved feedbacks
	if 'Open' in import_issue_status:
		sql = "SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,B1.Comment_Reviewer,B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time FROM BoardReview B1 WHERE B1.BoardID = %s AND B1.ComponentID = %s AND B1.Submitted = %s AND (((B1.AreaOfIssue <> '' AND B1.AreaOfIssue IS NOT NULL) AND ((B1.IssueStatus <> %s AND B1.IssueStatus <> %s) OR (B1.IssueStatus IS NULL))) OR (B1.IssueStatus IN %s)) ORDER BY B1.CommentID,B1.Submitted"
		val = (import_boardid,comp_id,'yes','Close','Conditional Waiver',import_issue_status)
		parents = execute_query(sql,val)

	else:
		sql = "SELECT DISTINCT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,B1.Comment_Reviewer,B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time FROM BoardReview B1 WHERE B1.BoardID = %s AND B1.ComponentID = %s AND B1.Submitted = %s AND B1.IssueStatus IN %s ORDER BY B1.CommentID,B1.Submitted"
		val = (import_boardid,comp_id,'yes',import_issue_status)
		parents = execute_query(sql,val)

	result = []
	import_feedbacks = []

	for i in parents:
		result.append(i)

	f_no = 0


	for i in range(0,len(result)):

		f_row_span = 1
		f_no_td_enabled = True

		if result[i][12] == 0:
			f_no += 1							

		temp = []
		temp.append(result[i][0])
		temp.append(result[i][1])
		temp.append(result[i][2])
		#temp.append(0)

		temp.append(f_no_td_enabled)
		temp.append(f_row_span)
		temp.append(f_no)

		temp.append(result[i][24])
		temp.append(result[i][3])

		if result[i][4] is not None:
			temp.append(result[i][4].replace("&"," and "))
		else:
			temp.append('')

		temp.append(result[i][5])

		if result[i][6] is not None:
			temp.append(result[i][6].replace("&"," and "))
		else:
			temp,append('')

		temp.append(result[i][7])

		if result[i][23] is not None:
			temp.append(result[i][23].replace("&"," and "))
		else:
			temp.append('')

		#download attachement
		temp.append(result[i][26])

		temp.append(result[i][8])
		#temp.append(result[i][9])

		if result[i][9] is not None:
			temp_design_comments = result[i][9].replace("---------","")

			if temp_design_comments.find("--") != -1:
				temp1_design_comments = temp_design_comments.split("--",1)
				temp.append(temp1_design_comments[0])
			else:
				temp.append(temp_design_comments)
		else:
			temp.append('')

		temp.append(result[i][27])

		temp.append(result[i][19])

		temp.append(result[i][20])

		temp.append(result[i][21])

		temp.append('') #bgcolor

		temp.append(result[i][24]) # import issues filename (Design Document Name)

		import_feedbacks.append(temp)

	for i in range(0,len(import_feedbacks)):

		f_row_span = 0
		import_feedbacks[i][3] = True

		for j in range(i,len(import_feedbacks)):
			if str(import_feedbacks[i][5]) == str(import_feedbacks[j][5]):
				f_row_span += 1

		import_feedbacks[i][4] = copy.deepcopy(f_row_span)

		if i>0:
			if str(import_feedbacks[i][5]) == str(import_feedbacks[i-1][5]):
				import_feedbacks[i][3] = False
				import_feedbacks[i][20] = 'bgcolor="#effbff"'
				import_feedbacks[i-1][20] = 'bgcolor="#effbff"'

	# convert python to javascript json format
	data["import_feedbacks"] = copy.deepcopy(import_feedbacks)

	sql = "SELECT AreaofIssue FROM AreaOfIssue ORDER BY AreaofIssue"
	area = execute_query_sql(sql)

	areas=[]
	for i in area:
		areas.append(i[0])

	data["areas"] = areas

	return jsonify(data)


@app.route("/submit_feedbacks_data",methods = ['POST', 'GET'])
def submit_feedbacks_data():

	wwid = session.get('wwid')
	username = session.get('username')

	data = {}
	data["comment_id_list"] = []

	board_id = request.form.get("board_id")
	comp_id = request.form.get("comp_id")
	comp_selected_list = eval(request.form.get("comp_select[]"))
	new_issues_count = int(request.form.get("new_issues_count_"+str(comp_id)))
	issues_count = int(request.form.get("issues_count_"+str(comp_id)))

	is_signoff = request.form.get("is_signoff_"+str(comp_id))

	pdg = request.form.get("pdg_"+str(comp_id))
	pdg_comments = request.form.get("pdg_comments_elect_"+str(comp_id))

	area_of_quality_issue = request.form.get("area_of_quality_issue_"+str(comp_id))
	basic_quality_comments = request.form.get("basic_quality_comments_"+str(comp_id))

	is_submit = False
	is_submitted = None
	if request.form.get("is_submit_"+str(comp_id)) == "yes":
		is_submit = True
		is_submitted = "yes"

	is_electric_submitted = False
	is_design_submitted = False
	is_reviewer2_submitted = False

	is_rev0p6_design = False
	is_rev1p0_design = True
	board_name = ""

	newly_added_feedbacks = []
	edited_feedbacks = []
	newly_added_feedbacks_mail = ''
	edited_feedbacks_mail = ''

	current_time = datetime.datetime.now(tz)
	current_date =  datetime.datetime.now(tz).strftime('%Y-%m-%d')

	query = "SELECT BoardName,ReviewTimelineID FROM BoardDetails WHERE BoardID=%s"
	val=(board_id,)
	board_dts_rs=execute_query(query,val)

	if board_dts_rs != ():
		board_name = board_dts_rs[0][0]

		if board_dts_rs[0][1] in [1,'1']:
			is_rev0p6_design = True
			is_rev1p0_design = False

	query = "SELECT ComponentName FROM ComponentType WHERE ComponentID=%s"
	val=(comp_id,)
	comp_name=execute_query(query,val)[0][0]

	# log table
	try:
		if is_submit:
			log_notes = 'User has Submitted Feedbacks details for <br>Design ID: '+str(board_id)
		else:
			log_notes = 'User has Saved Feedbacks details for <br>Design ID: '+str(board_id)

		log_notes += '<br>Component Name: '+str(comp_id)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Feedbacks',board_id,0,comp_id,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	pdg_submit = None
	if is_submit:
		pdg_submit = "yes"

		sql = "SELECT * FROM ScheduleTableComponent WHERE BoardID = %s AND ComponentID = %s AND ScheduleStatusID = %s"
		val = (board_id,comp_id,3)
		rs_delete_saved_fb = execute_query(sql, val)

		if rs_delete_saved_fb != ():
			sql = "DELETE FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND AddedByDesignTeam = %s AND Saved = %s AND (Submitted = %s OR Submitted IS NULL)"
			val = (board_id,comp_id,"yes","yes","no")
			execute_query(sql,val)		


		sql = "UPDATE ScheduleTableComponent SET ScheduleStatusID = %s WHERE BoardID = %s AND ComponentID = %s"
		val = (2,board_id,comp_id)
		execute_query(sql,val)		

		sql = "UPDATE BoardReviewDesigner SET IsPdgElectricalSubmitted = %s WHERE BoardID = %s AND ComponentID = %s"
		val = (pdg_submit,board_id,comp_id)
		execute_query(sql,val)

	if pdg is not None:

		sql = "UPDATE BoardReviewDesigner SET PDG_Electrical = %s,CommentElectrical = %s,CommntElectricalUpdatedBy = %s WHERE BoardID = %s AND ComponentID = %s"
		val = (pdg,pdg_comments,wwid,board_id,comp_id)
		execute_query(sql,val)

	# to update Risk level for electrical owners
	sql = "SELECT CommentID,FeedbackNo FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND RiskLevelSubmitted IS NULL"
	val = (board_id,comp_id)
	rs_risk_level_comment_id = execute_query(sql, val)


	if rs_risk_level_comment_id != ():
		for i in range(0,len(rs_risk_level_comment_id)):

			try:
				#risk_level_update = copy.deepcopy(data["risk_level_add_"+str(comp_id)+"_"+str(rs_risk_level_comment_id[i][0])])
				risk_level_update = request.form.get("risk_level_add_"+str(comp_id)+"_"+str(rs_risk_level_comment_id[i][0]))

				if risk_level_update is not None:

					sql = "UPDATE BoardReview SET RiskLevel = %s, RiskLevelSubmitted = %s, WWIDreviewer = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (risk_level_update,is_submitted,wwid,board_id,comp_id,rs_risk_level_comment_id[i][0])
					execute_query(sql,val)

					if is_submit:
						is_electric_submitted = True

					if rs_risk_level_comment_id[i][1] != 0:
						newly_added_feedbacks.append(rs_risk_level_comment_id[i][1])

			except Exception as inst:
				print(inst)
				pass

	# to update "to be filled by design owners part"
	sql = "SELECT CommentID,FeedbackNo FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND ((Submitted_Designer IS NULL) OR (Submitted_Designer <> 'yes'))"
	val = (board_id,comp_id)
	rs_designer_comment_id = execute_query(sql, val)


	if rs_designer_comment_id != ():
		for i in range(0,len(rs_designer_comment_id)):

			try:
				ImplementationStatus = request.form.get("impl_status_add_"+str(comp_id)+"_"+str(rs_designer_comment_id[i][0]))
				Comment = request.form.get("comment_add_"+str(comp_id)+"_"+str(rs_designer_comment_id[i][0]))

				try:
					designer_file = request.files["designer_file_"+str(comp_id)+"_"+str(rs_designer_comment_id[i][0])]
					DesignerFileName = designer_file.filename

				except:
					designer_file = None
					DesignerFileName = 'No File'

				#if designer_file.filename not in ['',None]:
				if designer_file is not None:
					if designer_file.filename != '':
						file=designer_file.read()
						filename=designer_file.filename
						fname = board_name+"_"+comp_name+"_"+str(rs_designer_comment_id[i][0])+"_"+filename

						sql = "INSERT INTO FileStorage (CommentID,DesignerFilename,DesignerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE DesignerFilename = %s, DesignerFile = %s"
						val = (rs_designer_comment_id[i][0],fname,file,fname,file)
						execute_query(sql,val)

						sql = "UPDATE BoardReview SET DesignerFileName = %s, WWIDdesigner = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (filename,wwid,board_id,comp_id,rs_designer_comment_id[i][0])
						execute_query(sql,val)						

				if ImplementationStatus is not None:

					is_edit_save_flag_design_value = 1
					if is_submit:
						Comment += "--"+str(username)+", Date: "+str(current_date)
						is_edit_save_flag_design_value = 0

					sql = "UPDATE BoardReview SET ImplementationStatus = %s, Comment = %s, Saved_Designer = %s, Submitted_Designer = %s, WWIDdesigner = %s, Submit_Time = %s, is_edit_save_flag_design = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (ImplementationStatus,Comment,"yes",is_submitted,wwid,t,is_edit_save_flag_design_value,board_id,comp_id,rs_designer_comment_id[i][0])
					execute_query(sql,val)

					if is_submit:
						is_design_submitted = True

					if rs_designer_comment_id[i][1] != 0:
						newly_added_feedbacks.append(rs_designer_comment_id[i][1])

					# to update UploadSignOffFiles files to reflect updated file to electrical owner's only on submit
					if is_submit:
						sql = "SELECT * FROM UploadSignOffFilesTemp WHERE BoardID = %s AND ComponentID = %s"
						val = (board_id,comp_id)
						rs_signoff_files_temp = execute_query(sql, val)
						print("rs_signoff_files_temp: ",rs_signoff_files_temp)

						if rs_signoff_files_temp != ():
							print("enter")
							# first delete and then copy from temp table, once copied, then delete from temp table
							sql = "DELETE FROM UploadSignOffFiles WHERE BoardID = %s AND ComponentID = %s"
							val = (board_id,comp_id)
							rs_signoff_files_tempa = execute_query(sql, val)
							print("rs_signoff_files_tempa: ",rs_signoff_files_tempa)

							sql = "INSERT INTO UploadSignOffFiles SELECT * FROM UploadSignOffFilesTemp WHERE BoardID = %s AND ComponentID = %s"
							val = (board_id,comp_id)
							rs_temp = execute_query(sql, val)
							print("rs_temp: ",rs_temp)

							sql = "DELETE FROM UploadSignOffFilesTemp WHERE BoardID = %s AND ComponentID = %s"
							val = (board_id,comp_id)
							rs_signoff_files_tempb = execute_query(sql, val)
							print("rs_signoff_files_tempb: ",rs_signoff_files_tempb)

			except Exception as inst:
				print(inst)
				pass

	# to update "to be filled by Electrial owner reviewer part"
	sql = "SELECT CommentID,FeedbackNo FROM BoardReview WHERE BoardID = %s AND ComponentID = %s"
	val = (board_id,comp_id)
	rs_reviewer2_comment_id = execute_query(sql, val)


	if rs_reviewer2_comment_id != ():

		for i in range(0,len(rs_reviewer2_comment_id)):

			try:
				issue_status = request.form.get("issue_status_add_"+str(comp_id)+"_"+str(rs_reviewer2_comment_id[i][0]))
				signoff_comments = request.form.get("signoff_comments_add_"+str(comp_id)+"_"+str(rs_reviewer2_comment_id[i][0]))
				risk_level_signoff = request.form.get("risk_level_signoff_add_"+str(comp_id)+"_"+str(rs_reviewer2_comment_id[i][0]))

				if issue_status is not None:

					is_submitted_signoff_sec = copy.deepcopy(is_submitted)

					if (issue_status == "") or (signoff_comments == "") or (risk_level_signoff == ""):
						is_submitted_signoff_sec = None

					# check for chilld issues submit
					sql = "SELECT CommentID FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND ActualParentID = %s ORDER BY CommentID DESC"
					val = (board_id,comp_id,rs_reviewer2_comment_id[i][0])
					rs_orginal_comment_id = execute_query(sql, val)

					temp_comment_id = copy.deepcopy(rs_reviewer2_comment_id[i][0])
					if rs_orginal_comment_id != ():
						temp_comment_id = copy.deepcopy(rs_orginal_comment_id[0][0])

					if (is_submit) and (is_submitted_signoff_sec is not None):
						signoff_comments += "--"+str(username)+", Date: "+str(current_date)

					sql = "UPDATE BoardReview SET IssueStatus = %s, Comment_Reviewer = %s, RiskLevelSignOff = %s, Saved_Reviewer2 = %s, Submitted_Reviewer2 = %s, WWIDreviewer = %s, Submit_Time = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (issue_status,signoff_comments,risk_level_signoff,"yes",is_submitted_signoff_sec,wwid,t,board_id,comp_id,temp_comment_id)
					execute_query(sql,val)

					if is_submit:
						is_reviewer2_submitted = True

					#if rs_reviewer2_comment_id[i][1] != 0:
					#	newly_added_feedbacks.append(rs_reviewer2_comment_id[i][1])

			except Exception as inst:
				print(inst)
				pass



	for j in range(0,issues_count):

		try:
			comment_id = request.form.get("comment_id_"+str(comp_id)+"_"+str(j))
			import_comment_id = request.form.get("import_comment_id_"+str(comp_id)+"_"+str(j))
			is_imported_form = request.form.get("is_imported_"+str(comp_id)+"_"+str(j))
			imported_from = request.form.get("imported_from_"+str(comp_id)+"_"+str(j))
			is_valid = True
		except Exception as inst:
			is_valid = False

		try:
			design_doc_name = request.form.get("design_doc_name_"+str(comp_id)+"_"+str(j))
			design_doc_type = request.form.get("design_doc_type_"+str(comp_id)+"_"+str(j))
			signal_name = request.form.get("signal_name_"+str(comp_id)+"_"+str(j))
			area_of_issue = request.form.get("area_of_issue_"+str(comp_id)+"_"+str(j))
			feedback_summary = request.form.get("feedback_summary_"+str(comp_id)+"_"+str(j))
			risk_level = request.form.get("risk_level_"+str(comp_id)+"_"+str(j))
			feedback_ref = request.form.get("feedback_ref_"+str(comp_id)+"_"+str(j))
			is_valid_electrical = True
		except Exception as inst:
			print(inst)
			is_valid_electrical = False
		
		try:
			electrical_file = request.files["electrical_file_"+str(comp_id)+"_"+str(j)]
			ReviewerFileName = electrical_file.filename
		except:
			electrical_file = None
			ReviewerFileName = 'No File'

		designer_file = None
		DesignerFileName = 'No File'

		if is_valid_electrical:
			if (design_doc_name == "") and (design_doc_type == "") and (signal_name == "") and (area_of_issue == "") and (feedback_summary == "") and (risk_level == "") and (feedback_ref == ""):
				is_valid = False

		if is_valid:
			if comment_id is None:
				comment_id = 0

			if import_comment_id is None:
				import_comment_id = 0

			comment_id = int(comment_id)
			import_comment_id = int(import_comment_id)
			DesignerFeedbackGiven = None
			#Saved_Designer = None
			#Submitted_Designer = None
			Submitted_Electrical = None
			
			is_imported = None
			imported_from_design_id = None
			imported_by = None

			if is_imported_form == "yes":
				is_imported = 'yes'
				imported_from_design_id = int(imported_from)
				imported_by = int(wwid)

				# if import is yes, then import_comment_id is zero, then which means imported data are saved and later user has loading the data, 
				# so in this case files are already uploaded, so we can map to current comment id
				if import_comment_id == 0:
					import_comment_id = copy.deepcopy(comment_id)

				sql = "SELECT ReviewerFileName,DesignerFileName FROM BoardReview WHERE CommentID = %s"
				val = (import_comment_id,)
				rs_file_names = execute_query(sql, val)

				if rs_file_names != ():
					ReviewerFileName = rs_file_names[0][0]
					DesignerFileName = rs_file_names[0][1]

			if is_valid_electrical:
				if is_submit:
					Submitted_Electrical = "yes"


			if is_submit and is_valid_electrical:
				if is_imported == "yes":

					try:
						feedback_summary = feedback_summary.replace("---------","<br>")
						#feedback_summary += "--"+str(username)+", Date: "+str(current_date)
						temp1 = feedback_summary.split("--",1)
						temp2 = temp1[1].split(", Date: ",1)
						temp_feedback_content = copy.deepcopy(temp1[0])	# it has feedback summary content
						temp_updated_by = copy.deepcopy(temp2[0])	# it has original username who has provided feedback

					except:
						temp_feedback_content = copy.deepcopy(feedback_summary)
						temp_updated_by = ''

					sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
					val = (imported_from_design_id,)
					imported_from_design_name = execute_query(sql, val)[0][0]

					feedback_summary = temp_feedback_content+'<br><br>'+'<span style="color:grey; font-size: smaller;">Imported from: '+imported_from_design_name+' [ID:'+str(imported_from_design_id)+']'
					feedback_summary += '<br>Imported by: '+str(username)
					feedback_summary += '<br>Imported on: '+str(current_date)
					feedback_summary += '<br>Comments by: '+str(temp_updated_by)+'</span>'
				else:
					if feedback_summary is not None:
						feedback_summary += "--"+str(username)+", Date: "+str(current_date)

			files_upload_comment_id = copy.deepcopy(comment_id)

			if is_submit:
				# to get maximum feeback number for board and Interface
				sql = "SELECT IFNULL(MAX(FeedbackNo),0) FROM BoardReview WHERE BoardID = %s AND ComponentID = %s"
				val = (board_id,comp_id)
				max_feedback_no_rs = execute_query(sql, val)
				max_feedback_no = 1
				if max_feedback_no_rs != ():
					max_feedback_no = max_feedback_no_rs[0][0] + 1
			else:
				max_feedback_no = 0

			if comment_id == 0:

				if ((is_submit and (design_doc_type is not None)) or ((is_submit == False) and (signal_name is not None))):

					sql = "INSERT INTO BoardReview (BoardID,ComponentID,DesignDocument,SignalName,AreaOfIssue,FeedbackSummary,RiskLevel,ReviewerFeedbackGiven,DesignerFeedbackGiven,ParentCommentID,Saved,Submitted,ReferenceNumber,BoardFileName,ReviewerFileName,DesignerFileName,ActualParentID,WWIDreviewer,Submit_Time,AddedByDesignTeam,IsImported,ImportedFromDesignID,ImportedBy,RiskLevelSubmitted,UpdatedOnForDesignSection,FeedbackNo,is_edit_save_flag,is_edit_save_flag_design) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
					val = (board_id,comp_id,design_doc_type,signal_name,area_of_issue,feedback_summary,risk_level,"yes",DesignerFeedbackGiven,0,"yes",Submitted_Electrical,feedback_ref,design_doc_name,ReviewerFileName,DesignerFileName,0,wwid,current_time,None,is_imported,imported_from_design_id,imported_by,"yes",current_time,max_feedback_no,0,0)
					execute_query(sql,val)

					if is_submit:
						is_electric_submitted = True

					sql = "SELECT LAST_INSERT_ID()"
					comid =  execute_query_sql(sql)[0][0]

					# to have newly added / edited feedbacks number for email purpose
					sql = "SELECT IFNULL(FeedbackNo,'') FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (board_id,comp_id,comid)
					newly_added_feedbacks_rs = execute_query(sql, val)

					if newly_added_feedbacks_rs != ():
						if newly_added_feedbacks_rs[0][0] != '':
							newly_added_feedbacks.append(newly_added_feedbacks_rs[0][0])

					files_upload_comment_id = copy.deepcopy(comid)

					temp = ["comment_id_"+str(comp_id)+"_"+str(j),comid]

					data["comment_id_list"].append(temp)

					# for import alone
					if is_imported_form == "yes":

						if import_comment_id != 0:

							sql = "INSERT INTO FileStorage (CommentID,ReviewerFilename,ReviewerFile,DesignerFilename,DesignerFile,Reviewer2Filename,Reviewer2File) SELECT %s,b.ReviewerFilename,b.ReviewerFile,b.DesignerFilename,b.DesignerFile,b.Reviewer2Filename,b.Reviewer2File FROM FileStorage b WHERE b.CommentID = %s"
							val = (comid,import_comment_id)
							execute_query(sql,val)

			else:
				# update table
				update_feedback_no = 0
				if is_submit:
					is_electric_submitted = True

					# to get maximum feeback number for board and Interface
					sql = "SELECT IFNULL(FeedbackNo,0) FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (board_id,comp_id,comment_id)
					temp_feedbackno = execute_query(sql, val)

					if temp_feedbackno != ():
						if int(temp_feedbackno[0][0]) == 0:
							update_feedback_no = copy.deepcopy(max_feedback_no)
						else:
							update_feedback_no = copy.deepcopy(temp_feedbackno[0][0])

				val = []
				sql = "UPDATE BoardReview SET ImportedBy = ImportedBy"

				if is_valid_electrical:
					sql += " ,DesignDocument = %s,SignalName = %s,AreaOfIssue = %s,FeedbackSummary = %s,RiskLevel = %s,RiskLevelSubmitted = %s,ReviewerFeedbackGiven = %s,Saved = %s,Submitted = %s,ReferenceNumber = %s,BoardFileName = %s"
					val.append(design_doc_type)
					val.append(signal_name)
					val.append(area_of_issue)
					val.append(feedback_summary)
					val.append(risk_level)
					val.append("yes")
					val.append("yes")
					val.append("yes")
					val.append(Submitted_Electrical)
					val.append(feedback_ref)
					val.append(design_doc_name)

					if ReviewerFileName != '':
						sql += " ,ReviewerFileName = %s"
						val.append(ReviewerFileName)

				if (is_submit) and (update_feedback_no != 0):
					sql += " ,FeedbackNo = %s"
					val.append(update_feedback_no)

				sql += " WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s AND WWIDreviewer = %s"
				val.append(board_id)
				val.append(comp_id)
				val.append(comment_id)
				val.append(wwid)

				val = tuple(val)
				execute_query(sql,val)

				# to have newly added / edited feedbacks number for email purpose
				#sql = "SELECT IFNULL(FeedbackNo,'') FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
				#val = (board_id,comp_id,comment_id)
				sql = "SELECT IFNULL(FeedbackNo,''),is_edit_save_flag FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s AND WWIDreviewer = %s"
				val = (board_id,comp_id,comment_id,wwid)
				newly_added_feedbacks_rs = execute_query(sql, val)

				if newly_added_feedbacks_rs != ():
					if newly_added_feedbacks_rs[0][0] != '':
						if newly_added_feedbacks_rs[0][1] == 0:
							newly_added_feedbacks.append(newly_added_feedbacks_rs[0][0])
						else:
							edited_feedbacks.append(newly_added_feedbacks_rs[0][0])

				if is_submit:
					sql = "UPDATE BoardReview SET is_edit_save_flag = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s AND WWIDreviewer = %s"
					val = (0,board_id,comp_id,comment_id,wwid)
					execute_query(sql,val)

			# files upload
			if electrical_file is not None:
				if electrical_file.filename != '':
					file=electrical_file.read()
					filename=electrical_file.filename
					fname = board_name+"_"+comp_name+"_"+str(j)+"_"+filename

					sql = "INSERT INTO FileStorage (CommentID,ReviewerFilename,ReviewerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE ReviewerFilename = %s, ReviewerFile = %s"
					val = (files_upload_comment_id,fname,file,fname,file)
					execute_query(sql,val)


	# to update child feedback "to be filled by Electrial owner 1st part"
	sql = "SELECT CommentID,FeedbackNo FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND Submitted_Designer = %s"
	val = (board_id,comp_id,"yes")
	rs_child_comment_id = execute_query(sql, val)


	if rs_child_comment_id != ():
		for i in range(0,len(rs_child_comment_id)):

			try:
				child_issue_comment_id = int(request.form.get("child_issue_comment_id_"+str(comp_id)+"_"+str(rs_child_comment_id[i][0])))
				is_valid_child_issue = request.form.get("is_valid_child_issue_"+str(comp_id)+"_"+str(rs_child_comment_id[i][0]))
				child_design_doc_name = request.form.get("child_design_doc_name_add_"+str(comp_id)+"_"+str(rs_child_comment_id[i][0]))
				child_issue_summary = request.form.get("child_issue_summary_add_"+str(comp_id)+"_"+str(rs_child_comment_id[i][0]))
				child_risk_level = request.form.get("child_risk_level_add_"+str(comp_id)+"_"+str(rs_child_comment_id[i][0]))
				child_issue_ref = request.form.get("child_issue_ref_add_"+str(comp_id)+"_"+str(rs_child_comment_id[i][0]))

				if is_valid_child_issue == "yes":

					ReviewerFeedbackGiven = None
					ReviewerFileName = None

					if is_submit:
						child_issue_summary += "--"+str(username)+", Date: "+str(current_date)
						ReviewerFeedbackGiven = "yes"

					if child_issue_comment_id == 0:
						sql = "SELECT DesignDocument,SignalName,AreaOfIssue,ActualParentID FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (board_id,comp_id,rs_child_comment_id[i][0])
						rs_child_parent_details = execute_query(sql, val)

						if str(rs_child_parent_details[0][3]) == '0':
							actual_parent_commnet_id = rs_child_comment_id[i][0]
						else:
							actual_parent_commnet_id = rs_child_parent_details[0][3]

						sql = "INSERT INTO BoardReview (BoardID,ComponentID,DesignDocument,SignalName,AreaOfIssue,FeedbackSummary,RiskLevel,ReviewerFeedbackGiven,ParentCommentID,Saved,Submitted,ReferenceNumber,BoardFileName,ReviewerFileName,ActualParentID,WWIDreviewer,Submit_Time,RiskLevelSubmitted,UpdatedOnForDesignSection,FeedbackNo,is_edit_save_flag,is_edit_save_flag_design) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
						val = (board_id,comp_id,rs_child_parent_details[0][0],rs_child_parent_details[0][1],rs_child_parent_details[0][2],child_issue_summary,child_risk_level,ReviewerFeedbackGiven,rs_child_comment_id[i][0],"yes",is_submitted,child_issue_ref,child_design_doc_name,ReviewerFileName,actual_parent_commnet_id,wwid,t,is_submitted,t,rs_child_comment_id[i][1],0,0)
						execute_query(sql,val)

						sql = "SELECT LAST_INSERT_ID()"
						comid =  execute_query_sql(sql)[0][0]

						# to have newly added / edited feedbacks number for email purpose
						sql = "SELECT IFNULL(FeedbackNo,'') FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (board_id,comp_id,comid)
						newly_added_feedbacks_rs = execute_query(sql, val)

						if newly_added_feedbacks_rs != ():
							if newly_added_feedbacks_rs[0][0] != '':
								newly_added_feedbacks.append(newly_added_feedbacks_rs[0][0])

						temp = ["child_issue_comment_id_"+str(comp_id)+"_"+str(rs_child_comment_id[i][0]),comid]
						data["comment_id_list"].append(temp)

						# updating has child info for parent feedback
						sql = "UPDATE BoardReview SET HasChild = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = ("yes",board_id,comp_id,rs_child_comment_id[i][0])
						execute_query(sql,val)

					else:
						sql = "UPDATE BoardReview SET FeedbackSummary = %s,RiskLevel = %s,ReviewerFeedbackGiven = %s,Submitted = %s,ReferenceNumber = %s,BoardFileName = %s, Submit_Time = %s,RiskLevelSubmitted = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s AND WWIDreviewer = %s"
						val = (child_issue_summary,child_risk_level,ReviewerFeedbackGiven,is_submitted,child_issue_ref,child_design_doc_name,t,is_submitted,board_id,comp_id,child_issue_comment_id,wwid)
						execute_query(sql,val)

						# to have newly added / edited feedbacks number for email purpose
						sql = "SELECT IFNULL(FeedbackNo,''),is_edit_save_flag FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s AND WWIDreviewer = %s"
						val = (board_id,comp_id,child_issue_comment_id,wwid)
						newly_added_feedbacks_rs = execute_query(sql, val)

						if newly_added_feedbacks_rs != ():
							if newly_added_feedbacks_rs[0][0] != '':
								if newly_added_feedbacks_rs[0][1] == 0:
									newly_added_feedbacks.append(newly_added_feedbacks_rs[0][0])
								else:
									edited_feedbacks.append(newly_added_feedbacks_rs[0][0])

						if is_submit:
							sql = "UPDATE BoardReview SET is_edit_save_flag = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s AND WWIDreviewer = %s"
							val = (0,board_id,comp_id,child_issue_comment_id,wwid)
							execute_query(sql,val)

					# to update child issue file details
					try:
						electrical_file = request.files["child_electrical_file_"+str(comp_id)+"_"+str(rs_child_comment_id[i][0])]
						ReviewerFileName = electrical_file.filename

						if electrical_file is not None:
							if electrical_file.filename != '':
								file=electrical_file.read()
								filename=electrical_file.filename

								if child_issue_comment_id == 0:
									child_comid = copy.deepcopy(comid)
								else:
									child_comid = copy.deepcopy(child_issue_comment_id)

								fname = board_name+"_"+comp_name+"_"+str(child_comid)+"_"+filename

								sql = "INSERT INTO FileStorage (CommentID,ReviewerFilename,ReviewerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE ReviewerFilename = %s, ReviewerFile = %s"
								val = (child_comid,fname,file,fname,file)
								execute_query(sql,val)

								sql = "UPDATE BoardReview SET ReviewerFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
								val = (filename,board_id,comp_id,child_comid)
								execute_query(sql,val)						

					except Exception as inst:
						print(inst)

					if is_submit:
						is_electric_submitted = True

			except Exception as inst:
				print(inst)
				pass

	is_edited_electrical = False
	is_edited_designer = False
	is_edited_reviewer2 = False

	# to update for edit feedbacks
	sql = "SELECT CommentID,ParentCommentID,FeedbackNo FROM BoardReview WHERE BoardID = %s AND ComponentID = %s"
	val = (board_id,comp_id)
	rs_edit_comment_id = execute_query(sql, val)


	if rs_edit_comment_id != ():
		for i in range(0,len(rs_edit_comment_id)):

			try:
				edit_design_doc_name = request.form.get("edit_design_doc_name_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
				edit_design_doc_type = request.form.get("edit_design_doc_type_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
				edit_signal_name = request.form.get("edit_signal_name_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
				edit_area_of_issue = request.form.get("edit_area_of_issue_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
				edit_feedback_summary = request.form.get("edit_feedback_summary_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
				edit_risk_level = request.form.get("edit_risk_level_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
				edit_feedback_ref = request.form.get("edit_feedback_ref_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))

				edit_impl_status = request.form.get("edit_impl_status_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
				edit_comment = request.form.get("edit_comment_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))

				edit_issue_status = request.form.get("edit_issue_status_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
				edit_signoff_comments = request.form.get("edit_signoff_comments_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))
				edit_risk_level_signoff = request.form.get("edit_risk_level_signoff_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0]))

				if ((edit_design_doc_name is not None) or (edit_impl_status is not None) or (edit_issue_status is not None)):
					if is_submit:
						sql = "SELECT * FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (board_id,comp_id,rs_edit_comment_id[i][0])
						rs_temp = execute_query(sql,val)

						if rs_temp != ():
							sql = "UPDATE BoardReview SET EditDiscardFlag = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
							val = (0,board_id,comp_id,rs_edit_comment_id[i][0])
							execute_query(sql,val)

							sql = "DELETE FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
							val = (board_id,comp_id,rs_edit_comment_id[i][0])
							execute_query(sql,val)

					else:
						# save part
						# to backup submitted feedback data before saving edit data, to keep it for discard option for editing submitted data
						sql = "SELECT * FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (board_id,comp_id,rs_edit_comment_id[i][0])
						rs_temp = execute_query(sql,val)

						if rs_temp == ():	# for first time save only we are backing up submitted data
							sql = "INSERT INTO BoardReviewTemp SELECT * FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
							val = (board_id,comp_id,rs_edit_comment_id[i][0])
							execute_query(sql,val)

							sql = "UPDATE BoardReview SET EditDiscardFlag = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
							val = (1,board_id,comp_id,rs_edit_comment_id[i][0])
							execute_query(sql,val)

				# 1 st part
				if edit_design_doc_name is not None:

					if is_submit:
						if edit_feedback_summary is not None:
							edit_feedback_summary += '<br><br><span style="color:grey; font-size: smaller;">Edited By: '+str(username)+'<br>Edited On: '+str(current_date)+'</span>'

					sql = "UPDATE BoardReview SET BoardFileName = %s, FeedbackSummary = %s, RiskLevel = %s, ReferenceNumber = %s, Saved = %s, Submitted = %s, ReviewerFeedbackGiven = %s,WWIDreviewer = %s, Submit_Time = %s,RiskLevelSubmitted = %s,is_edit_save_flag = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (edit_design_doc_name,edit_feedback_summary,edit_risk_level,edit_feedback_ref,"yes",is_submitted,is_submitted,wwid,t,is_submitted,1,board_id,comp_id,rs_edit_comment_id[i][0])
					execute_query(sql,val)

					is_edited_electrical = True

					if rs_edit_comment_id[i][2] != 0:
						edited_feedbacks.append(rs_edit_comment_id[i][2])

					# for child feedback check
					if edit_design_doc_type is not None:

						sql = "UPDATE BoardReview SET DesignDocument = %s, SignalName = %s, AreaOfIssue = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
						val = (edit_design_doc_type,edit_signal_name,edit_area_of_issue,board_id,comp_id,rs_edit_comment_id[i][0])
						execute_query(sql,val)		


					# to update edit file attachement for electrical 1sr part
					try:
						electrical_file = request.files["edit_electrical_file_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])]
						ReviewerFileName = electrical_file.filename

						if electrical_file is not None:
							if electrical_file.filename != '':
								file=electrical_file.read()
								filename=electrical_file.filename
								fname = board_name+"_"+comp_name+"_"+str(comp_id)+"_"+filename

								sql = "INSERT INTO FileStorage (CommentID,ReviewerFilename,ReviewerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE ReviewerFilename = %s, ReviewerFile = %s"
								val = (rs_edit_comment_id[i][0],fname,file,fname,file)
								execute_query(sql,val)

								sql = "UPDATE BoardReview SET ReviewerFileName = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
								val = (filename,board_id,comp_id,rs_edit_comment_id[i][0])
								execute_query(sql,val)						

					except Exception as inst:
						print(inst)

					if is_submit:
						is_electric_submitted = True

				# 2nd part
				if edit_impl_status is not None:
					
					is_edit_save_flag_design_value = 1
					if is_submit:
						#edit_comment += "--"+str(username)+", Date: "+str(current_date)
						edit_comment += '<br><br><span style="color:grey; font-size: smaller;">Edited By: '+str(username)+'<br>Edited On: '+str(current_date)+'</span>'
						is_edit_save_flag_design_value = 0
						is_design_submitted = True

					sql = "UPDATE BoardReview SET ImplementationStatus = %s, Comment = %s, DesignerFeedbackGiven = %s, Saved_Designer = %s, Submitted_Designer = %s, WWIDdesigner = %s, Submit_Time = %s, is_edit_save_flag_design = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (edit_impl_status,edit_comment,is_submitted,"yes",is_submitted,wwid,t,is_edit_save_flag_design_value,board_id,comp_id,rs_edit_comment_id[i][0])
					execute_query(sql,val)

					is_edited_designer = True

					if rs_edit_comment_id[i][2] != 0:
						edited_feedbacks.append(rs_edit_comment_id[i][2])

					# to update edit file attachement for electrical 1sr part
					try:
						designer_file = request.files["edit_designer_file_"+str(comp_id)+"_"+str(rs_edit_comment_id[i][0])]
						DesignerFileName = designer_file.filename

						if designer_file is not None:
							if designer_file.filename != '':
								file=designer_file.read()
								filename=designer_file.filename
								fname = board_name+"_"+comp_name+"_"+str(comp_id)+"_"+filename

								sql = "INSERT INTO FileStorage (CommentID,DesignerFilename,DesignerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE DesignerFilename = %s, DesignerFile = %s"
								val = (rs_edit_comment_id[i][0],fname,file,fname,file)
								execute_query(sql,val)

								sql = "UPDATE BoardReview SET DesignerFilename = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
								val = (filename,board_id,comp_id,rs_edit_comment_id[i][0])
								execute_query(sql,val)						

					except Exception as inst:
						print(inst)

				# 3rd part
				if edit_issue_status is not None:

					is_submitted_signoff_sec = copy.deepcopy(is_submitted)

					if (edit_issue_status == "") or (edit_signoff_comments == "") or (edit_risk_level_signoff == ""):
						is_submitted_signoff_sec = None

					if (is_submit) and (is_submitted_signoff_sec is not None):
						#edit_signoff_comments += "--"+str(username)+", Date: "+str(current_date)
						edit_signoff_comments += '<br><br><span style="color:grey; font-size: smaller;">Edited By: '+str(username)+'<br>Edited On: '+str(current_date)+'</span>'

						is_reviewer2_submitted = True

					sql = "UPDATE BoardReview SET IssueStatus = %s, Comment_Reviewer = %s, RiskLevelSignOff = %s, Saved_Reviewer2 = %s, Submitted_Reviewer2 = %s, WWIDreviewer = %s, Submit_Time = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = (edit_issue_status,edit_signoff_comments,edit_risk_level_signoff,"yes",is_submitted_signoff_sec,wwid,t,board_id,comp_id,rs_edit_comment_id[i][0])
					execute_query(sql,val)

					is_edited_reviewer2 = True

					#if rs_edit_comment_id[i][2] != 0:
					#	edited_feedbacks.append(rs_edit_comment_id[i][2])

			except Exception as inst:
				print(inst)
				pass

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (board_id,)
	sku_plat = execute_query(sql,val)

	# disable all email part for sing-off interface directly
	if is_signoff == "yes":
		is_electric_submitted = False
		is_design_submitted = False
		is_reviewer2_submitted = False

	# email part
	if is_electric_submitted:
		#getting the secondary wwid's from the componentreview from database
			
		sql = "select distinct SecondaryWWID from ComponentReview C2, BoardReviewDesigner B1, CategoryLeadTable C3,ComponentType C4 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.CategoryID = C3.CategoryID AND C3.SKUID = %s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s"
		val = (board_id,comp_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		sec_ele_wwid = execute_query(sql,val)

		sec_wwid = []
		for i in range(0,len(sec_ele_wwid)):
			ele = sec_ele_wwid[i]

			for j in range(0,len(ele)):
				sec_ele = ele[j][1:-1]
				spl = sec_ele.split(",")

				for k in range(0,len(spl)):
					sec_wwid.append(spl[k])

		sql = "select distinct SecondaryWWID FROM ComponentDesign C2,BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s"
		val = (board_id,comp_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		sec_des_wwid = execute_query(sql,val)

		des_wwid = []
		for i in range(0,len(sec_des_wwid)):
			des = sec_des_wwid[i]

			for j in range(0,len(des)):
				sec_des = des[j][1:-1]
				spl = sec_des.split(",")

				for k in range(0,len(spl)):
					des_wwid.append(spl[k])

		email_list = []

		newly_added_feedbacks = [int(x) for x in newly_added_feedbacks]
		edited_feedbacks = [int(x) for x in edited_feedbacks]

		newly_added_feedbacks = set(newly_added_feedbacks)
		newly_added_feedbacks = sorted(list(newly_added_feedbacks))
		newly_added_feedbacks = [str(x) for x in newly_added_feedbacks]

		if len(newly_added_feedbacks) > 0:
			newly_added_feedbacks_mail = '<br><b>New Feedbacks: </b>'
			newly_added_feedbacks_mail_temp = ', '.join(newly_added_feedbacks)
			newly_added_feedbacks_mail += copy.deepcopy(newly_added_feedbacks_mail_temp)

		edited_feedbacks = set(edited_feedbacks)
		edited_feedbacks = sorted(list(edited_feedbacks))
		edited_feedbacks = [str(x) for x in edited_feedbacks]

		if len(edited_feedbacks) > 0:
			edited_feedbacks_mail = '<br><br><b>Edited Feedbacks: </b>'
			edited_feedbacks_mail_temp = ', '.join(edited_feedbacks)
			edited_feedbacks_mail += copy.deepcopy(edited_feedbacks_mail_temp)

		sql = "SELECT DISTINCT H1.EmailID,H4.EmailID FROM BoardReviewDesigner B1,  ComponentReview C2, HomeTable H1, HomeTable H2,HomeTable H4, CategoryLeadTable C3 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID AND C2.CategoryID = C3.CategoryID AND C3.CategoryLeadWWID = H4.WWID AND C3.SKUID = %s AND C3.PlatformID = %s  AND C3.MemTypeID = %s AND C3.DesignTypeID = %s "
		val = (board_id,comp_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],)
		emailids = execute_query(sql,val)

		for j in sec_wwid:
			sql = 'SELECT DISTINCT EmailID FROM HomeTable WHERE WWID =%s'
			val = (j,)
			eid1 = execute_query(sql,val)

			if(eid1 != ()):
				email_list.append(eid1[0][0])
		
		sql = "SELECT DISTINCT H1.EmailID FROM BoardReviewDesigner B1,  ComponentDesign C2, HomeTable H1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
		val = (board_id,comp_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		emailids2 = execute_query(sql,val)

		for j in des_wwid:
			sql = 'SELECT EmailID FROM HomeTable WHERE WWID =%s'
			val = (j,)
			eid1 = execute_query(sql,val)
			if(eid1 != ()):
				email_list.append(eid1[0][0])
		
		for k in emailids:
			email_list.append(k[0])
			email_list.append(k[1])

		for k in emailids2:
			email_list.append(k[0])

		sql="SELECT  a.CategoryName,b.EmailID from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2 WHERE cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID =%s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PLatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID ORDER BY cr.ComponentID"
		val=(sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],comp_id,sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		catlead=execute_query(sql,val)
		if(catlead != ()):
			for i in catlead:
				email_list.append(i[1])		

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
		val = (board_id,)
		designlist = execute_query(query,val)
		designlead_list = []
		for i in range(len(designlist)):
			eid = designlist[0][1]
			email_list.append(eid)


		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
		val = (board_id,)
		cadlist = execute_query(query,val)
		cadlead_list = []
		for i in range(len(cadlist)):
			eid = cadlist[0][1]
			email_list.append(eid)
			
		sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
		val = ('yes',)
		admin_email = execute_query(sql,val)
		for k in admin_email:
			email_list.append(k[0])


		sql = "SELECT PDG_Electrical FROM BoardReviewDesigner WHERE BoardID = %s AND ComponentID = %s "
		val = (board_id,comp_id)
		PDG = execute_query(sql,val)[0][0]	

		email_list = sorted(set(email_list), reverse=True)

		submitted_or_edited_text = "Submitted"
		if is_edited_electrical:
			submitted_or_edited_text = "Edited"

		subject="[ID:"+board_id+"] "+ comp_name+" - Feedback "+submitted_or_edited_text+" By Electrical Team"
		message=''' ERAM Design ID: ''' + board_id +" - "+board_name +''' <br>
				'''+comp_name + ''' - Feedback '''+submitted_or_edited_text+''' By : ''' + username + '''<br>'''

		if pdg is not None:
			message += '''PDG Compliance By Electrical Team:''' + pdg + ''' <br>'''

		message += '''
					Please proceed to visit https://eram.apps1-fm-int.icloud.intel.com/ to view '''+submitted_or_edited_text+''' feedback. <br><br>

					<b><u>AR to Design Team</u></b> <br><br>

					To provide comments / upload Sign-Off files: My Dashboard >> Feedback Submission Module >> '''+board_name+'''<br><br>
					'''+newly_added_feedbacks_mail+edited_feedbacks_mail+'''<br><br><br>Thanks, <br>ERAM.'''

		for m in email_list:
			send_mail(m,subject,message,email_list)
		#send_mail(reciever=', '.join(email_list),subject=subject,message=message)

		# basic design quality check 
		sql = "SELECT CommentID,IsSubmitted FROM BasicDesignQualityCheck WHERE BoardID = %s AND ComponentID = %s ORDER BY CommentID DESC"
		val = (board_id,comp_id)
		rs_temp = execute_query(sql,val)

		valid_email_trigger = False

		if rs_temp != ():

			for row in rs_temp:

				if row[1] in [None,"no",""]:

					sql = "UPDATE BasicDesignQualityCheck SET QualityCheck = %s,AreaOfQualityIssue = %s,Comments = %s,IsSubmitted = %s,UpdatedBy = %s,UpdatedOn = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
					val = ("met",0,"","yes",wwid,t,board_id,comp_id,row[0])
					execute_query(sql,val)

	# mail for designer part
	if is_design_submitted:
		sql = "select distinct SecondaryWWID from ComponentReview C2, BoardReviewDesigner B1, CategoryLeadTable C3,ComponentType C4 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.CategoryID = C3.CategoryID AND C3.SKUID = %s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s"
		val = (board_id,comp_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		sec_ele_wwid = execute_query(sql,val)

		sec_wwid = []
		for i in range(0,len(sec_ele_wwid)):
			ele = sec_ele_wwid[i]
			for j in range(0,len(ele)):
				sec_ele = ele[j][1:-1]
				spl = sec_ele.split(",")
				for k in range(0,len(spl)):
					sec_wwid.append(spl[k])

		sql = "select distinct SecondaryWWID FROM ComponentDesign C2,BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s"
		val = (board_id,comp_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		sec_des_wwid = execute_query(sql,val)

		des_wwid = []
		for i in range(0,len(sec_des_wwid)):
			des = sec_des_wwid[i]
			for j in range(0,len(des)):
				sec_des = des[j][1:-1]
				spl = sec_des.split(",")
				for k in range(0,len(spl)):
					des_wwid.append(spl[k])

		email_list = []

		sql = "SELECT DISTINCT H1.EmailID,H4.EmailID FROM BoardReviewDesigner B1,  ComponentReview C2, HomeTable H1, HomeTable H2,HomeTable H4, CategoryLeadTable C3 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID AND C2.CategoryID = C3.CategoryID AND C3.CategoryLeadWWID = H4.WWID AND C3.SKUID = %s AND C3.PlatformID = %s  AND C3.MemTypeID = %s AND C3.DesignTypeID = %s "
		val = (board_id,comp_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],)
		emailids = execute_query(sql,val)

		for j in sec_wwid:
			sql = 'SELECT DISTINCT EmailID FROM HomeTable WHERE WWID =%s'
			val = (j,)
			eid1 = execute_query(sql,val)
			if(eid1 != ()):
				email_list.append(eid1[0][0])
		
		sql = "SELECT DISTINCT H1.EmailID FROM BoardReviewDesigner B1,  ComponentDesign C2, HomeTable H1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
		val = (board_id,comp_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		emailids2 = execute_query(sql,val)

		for j in des_wwid:
			sql = 'SELECT EmailID FROM HomeTable WHERE WWID =%s'
			val = (j,)
			eid1 = execute_query(sql,val)
			if(eid1 != ()):
				email_list.append(eid1[0][0])

		wwid = session.get('wwid')
		name = session.get('username')
		for k in emailids:
			email_list.append(k[0])
			email_list.append(k[1])

		for k in emailids2:
			email_list.append(k[0])
				
		sql="SELECT  a.CategoryName,b.EmailID from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2 WHERE cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID = %s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PLatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID ORDER BY cr.ComponentID"
		val=(sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],comp_id,sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		catlead=execute_query(sql,val)
		if(catlead != ()):
			for i in catlead:
				email_list.append(i[1])		


		sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
		val = ('yes',)
		admin_email = execute_query(sql,val)
		for k in admin_email:
			email_list.append(k[0])

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
		val = (board_id,)
		designlist = execute_query(query,val)
		designlead_list = []
		for i in range(len(designlist)):
			eid = designlist[0][1]
			email_list.append(eid)


		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
		val = (board_id,)
		cadlist = execute_query(query,val)
		cadlead_list = []
		for i in range(len(cadlist)):
			eid = cadlist[0][1]
			email_list.append(eid)


		email_list = sorted(set(email_list), reverse=True)
		subject="[ID:"+board_id+"] "+ comp_name+" - Sign-Off Files Uploaded By Design Team"
		message=''' ERAM Design ID: ''' + board_id +" - "+board_name +''' <br>
				'''+comp_name + ''' - Signoff Files and Comments updated By : ''' + name + '''<br>
					Please proceed to visit https://eram.apps1-fm-int.icloud.intel.com/ to close and Sign-Off feedback. <br><br>

					<u> <b>AR to Electrical Owner </b></u> <br>

					To provide Feedback/Sign-Off : My Dashboard >> Feedback Submission Module >> '''+board_name+'''<br><br>

					'''+newly_added_feedbacks_mail+edited_feedbacks_mail+'''<br><br><br>Thanks, <br>ERAM.'''
	
		for m in email_list:
			send_mail(m,subject,message,email_list)

	# mail for reviewer2 part
	if is_reviewer2_submitted and not is_electric_submitted:
		sql = "select distinct SecondaryWWID from ComponentReview C2, BoardReviewDesigner B1, CategoryLeadTable C3,ComponentType C4 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.CategoryID = C3.CategoryID AND C3.SKUID = %s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s"
		val = (board_id,comp_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		sec_ele_wwid = execute_query(sql,val)

		sec_wwid = []
		for i in range(0,len(sec_ele_wwid)):
			ele = sec_ele_wwid[i]
			for j in range(0,len(ele)):
				sec_ele = ele[j][1:-1]
				spl = sec_ele.split(",")
				for k in range(0,len(spl)):
					sec_wwid.append(spl[k])

		sql = "select distinct SecondaryWWID FROM ComponentDesign C2,BoardReviewDesigner B1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s"
		val = (board_id,comp_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		sec_des_wwid = execute_query(sql,val)

		des_wwid = []
		for i in range(0,len(sec_des_wwid)):
			des = sec_des_wwid[i]
			for j in range(0,len(des)):
				sec_des = des[j][1:-1]
				spl = sec_des.split(",")
				for k in range(0,len(spl)):
					des_wwid.append(spl[k])

		email_list = []

		sql = "SELECT DISTINCT H1.EmailID,H4.EmailID FROM BoardReviewDesigner B1,  ComponentReview C2, HomeTable H1, HomeTable H2,HomeTable H4, CategoryLeadTable C3 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID AND C2.CategoryID = C3.CategoryID AND C3.CategoryLeadWWID = H4.WWID AND C3.SKUID = %s AND C3.PlatformID = %s  AND C3.MemTypeID = %s AND C3.DesignTypeID = %s "
		val = (board_id,comp_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],)
		emailids = execute_query(sql,val)

		for j in sec_wwid:
			sql = 'SELECT DISTINCT EmailID FROM HomeTable WHERE WWID =%s'
			val = (j,)
			eid1 = execute_query(sql,val)
			if(eid1 != ()):
				email_list.append(eid1[0][0])
		
		sql = "SELECT DISTINCT H1.EmailID FROM BoardReviewDesigner B1,  ComponentDesign C2, HomeTable H1 WHERE B1.BoardID = %s AND B1.ComponentID = C2.ComponentID AND B1.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s  AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
		val = (board_id,comp_id,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		emailids2 = execute_query(sql,val)

		for j in des_wwid:
			sql = 'SELECT EmailID FROM HomeTable WHERE WWID =%s'
			val = (j,)
			eid1 = execute_query(sql,val)
			if(eid1 != ()):
				email_list.append(eid1[0][0])

		wwid = session.get('wwid')
		name = session.get('username')
		for k in emailids:
			email_list.append(k[0])
			email_list.append(k[1])

		for k in emailids2:
			email_list.append(k[0])
				
		sql="SELECT  a.CategoryName,b.EmailID from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2 WHERE cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID = %s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PLatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID ORDER BY cr.ComponentID"
		val=(sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],comp_id,sku_plat[0][0], sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
		catlead=execute_query(sql,val)
		if(catlead != ()):
			for i in catlead:
				email_list.append(i[1])			

		sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
		val = ('yes',)
		admin_email = execute_query(sql,val)
		for k in admin_email:
			email_list.append(k[0])

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
		val = (board_id,)
		designlist = execute_query(query,val)
		designlead_list = []
		for i in range(len(designlist)):
			eid = designlist[0][1]
			email_list.append(eid)


		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
		val = (board_id,)
		cadlist = execute_query(query,val)
		cadlead_list = []
		for i in range(len(cadlist)):
			eid = cadlist[0][1]
			email_list.append(eid)


		subject="[ID:"+board_id+"] "+ comp_name+" - Feedback Sign-Off Status Updated by Electrical team"
		message=''' ERAM Design ID: ''' + board_id +" - "+board_name +''' <br>
				'''+comp_name + ''' - Feedback Sign-Off status updated by : ''' + name + '''<br>
					Please proceed to visit https://eram.apps1-fm-int.icloud.intel.com/ to view updated feedback. <br><br>

					'''+newly_added_feedbacks_mail+edited_feedbacks_mail+'''<br><br><br>Thanks, <br>ERAM.'''
		
		email_list = sorted(set(email_list), reverse=True)
		for m in email_list:
			send_mail(m,subject,message,email_list)			


	# to replace the component details section in ajax
	if is_submit:

		sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
		val = (wwid,)
		has_admin_access = execute_query(sql,val)[0][0]

		is_admin = False
		if has_admin_access == "yes":
			is_admin = True

		sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
		val = (wwid,)
		role = execute_query(sql,val)[0][0]

		is_elec_owner = False
		if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
			is_elec_owner = True

		is_design_owner = False
		if(role == 3 or role == 5 or role == 6):
			is_design_owner = True

		is_layout_owner = False
		if(role == 7 or role == 8 or role == 9):
			is_layout_owner = True

		sql = "SELECT * FROM BoardDetails WHERE BoardID = %s AND (DesignLeadWWID = %s OR CADLeadWWID = %s)"	
		val = (board_id,wwid,wwid)		
		rs_design_layout_lead = execute_query(sql,val)

		is_design_layout_lead = False
		if rs_design_layout_lead != ():
			is_design_layout_lead = True


		try:
			sql = "SELECT ScheduleStatusID FROM ScheduleTable WHERE BoardID = %s"
			val = (board_id,)
			board_status = execute_query(sql,val)[0][0]
		except:
			board_status = 0

		comp_list = []
		comp_list = get_all_interfaces_feedbacks(boardid=board_id)[1]

		my_designs_id = []
		temp_data = []
		temp_data = get_my_designs_feedbacks()		

		for i in range(0,len(temp_data[1])):
			my_designs_id.append(temp_data[1][i][0])

		my_components_id = []

		temp_data = []
		# for design & Layout Lead, Managers - both My interface and All interface should be same and have edit access, as we dont have any mapping for design/layout lead and managers at backend properly
		if is_design_layout_lead:
			temp_data = get_all_interfaces_feedbacks(boardid=board_id)
		else:
			temp_data = get_my_interfaces_feedbacks(boardid=board_id)

		for i in range(0,len(temp_data[1])):
			my_components_id.append(temp_data[1][i][0])

		data = get_feedbacks_data_page(data=data,boardid=board_id,compid=comp_id,complist=comp_list,sku_plat=sku_plat,board_status=board_status,my_designs_id=my_designs_id,my_components_id=my_components_id)

		sql = "SELECT AreaofIssue FROM AreaOfIssue ORDER BY AreaofIssue"
		area = execute_query_sql(sql)

		areas=[]
		for i in area:
			areas.append(i[0])

		return render('feedbacks_files_div_data.html',is_rev0p6_design=is_rev0p6_design,is_rev1p0_design=is_rev1p0_design,data=data,areas=areas,boardid=board_id,comp_id=comp_id,comp_selected_list=comp_selected_list,is_admin=is_admin,is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner)

	return jsonify(data)

@app.route("/feedbacks_signoff_files_submit",methods = ['POST', 'GET'])
def feedbacks_signoff_files_submit():
	
	wwid=session.get('wwid')
	username = session.get('username')
	is_admin = session.get('is_admin')

	data = {}
	data = dict(request.form)

	board_id = data['board_id']
	comp_list = eval(data['comp_select[]'])

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	query = "SELECT * FROM ScheduleTable a WHERE a.BoardID = %s AND a.ScheduleStatusID IN (2,6)"
	val = (board_id,)
	rs_board_status = execute_query(query, val)

	if rs_board_status == ():
		return feedbacks(boardid=int(board_id),comp_selected_list=comp_list,my_designs="All",my_interfaces="All")

	sql = "SELECT * FROM BoardDetails WHERE BoardID = %s AND (DesignLeadWWID = %s OR CADLeadWWID = %s)"	
	val = (board_id,wwid,wwid)		
	rs_design_layout_lead = execute_query(sql,val)

	is_design_layout_lead = False
	if rs_design_layout_lead != ():
		is_design_layout_lead = True

	# log table
	try:
		log_notes = 'User has submitted Design Files for Review for Design ID: '+str(board_id)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Design Files',board_id,0,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	board_file_id = None
	board_filename = None
	schematics_file_id = None
	schematics_filename = None
	stackup_file_id = None
	stackup_filename = None
	lenght_report_file_id = None
	lenght_report_filename = None
	other_files_id = None
	other_filesname = None

	is_valid_files = False

	if request.files['board_file'].filename != '':

		board_file = request.files['board_file']
		board_filename = secure_filename(request.files['board_file'].filename)

		# attaching file for litepi process
		file_response = litepi_file_process(boardid=board_id,file_name=board_filename,file_upload=board_file)
		print("file_response: ",file_response)

		board_file.stream.seek(0)	# reset file stream pointer
		board_file_read = board_file.read()	# read bytes from the file to upload in DB

		sql = "INSERT INTO UploadFileStorage VALUES (%s,%s,%s)"
		val = (None,board_id,board_file_read)
		execute_query(sql,val)

		sql = "SELECT LAST_INSERT_ID()"
		board_file_id = execute_query_sql(sql)[0][0]

		sql = "INSERT INTO UploadFileStorageMisc VALUES (%s,%s,%s,%s)"
		val = (board_file_id,board_id,t,wwid)
		execute_query(sql,val)

		is_valid_files = True


	if request.files['schematics_file'].filename != '':
		print("good.")
		schematics_file = request.files['schematics_file'].read()
		schematics_filename = request.files['schematics_file'].filename

		sql = "INSERT INTO UploadFileStorage VALUES (%s,%s,%s)"
		val = (None,board_id,schematics_file)
		execute_query(sql,val)

		sql = "SELECT LAST_INSERT_ID()"
		schematics_file_id = execute_query_sql(sql)[0][0]

		sql = "INSERT INTO UploadFileStorageMisc VALUES (%s,%s,%s,%s)"
		val = (schematics_file_id,board_id,t,wwid)
		execute_query(sql,val)

		is_valid_files = True
		

	if request.files['stackup_file'].filename != '':
		stackup_file = request.files['stackup_file'].read()
		stackup_filename = request.files['stackup_file'].filename

		sql = "INSERT INTO UploadFileStorage VALUES (%s,%s,%s)"
		val = (None,board_id,stackup_file)
		execute_query(sql,val)

		sql = "SELECT LAST_INSERT_ID()"
		stackup_file_id = execute_query_sql(sql)[0][0]

		sql = "INSERT INTO UploadFileStorageMisc VALUES (%s,%s,%s,%s)"
		val = (stackup_file_id,board_id,t,wwid)
		execute_query(sql,val)

		is_valid_files = True


	if request.files['lenght_report_file'].filename != '':
		lenght_report_file = request.files['lenght_report_file'].read()
		lenght_report_filename = request.files['lenght_report_file'].filename

		sql = "INSERT INTO UploadFileStorage VALUES (%s,%s,%s)"
		val = (None,board_id,lenght_report_file)
		execute_query(sql,val)

		sql = "SELECT LAST_INSERT_ID()"
		lenght_report_file_id = execute_query_sql(sql)[0][0]

		sql = "INSERT INTO UploadFileStorageMisc VALUES (%s,%s,%s,%s)"
		val = (lenght_report_file_id,board_id,t,wwid)
		execute_query(sql,val)

		is_valid_files = True


	if request.files['other_files'].filename != '':
		other_files = request.files['other_files'].read()
		other_filesname = request.files['other_files'].filename

		sql = "INSERT INTO UploadFileStorage VALUES (%s,%s,%s)"
		val = (None,board_id,other_files)
		execute_query(sql,val)

		sql = "SELECT LAST_INSERT_ID()"
		other_files_id = execute_query_sql(sql)[0][0]

		sql = "INSERT INTO UploadFileStorageMisc VALUES (%s,%s,%s,%s)"
		val = (other_files_id,board_id,t,wwid)
		execute_query(sql,val)

		is_valid_files = True


	# if no files are uploaded, then ignore all further update and email process
	if not is_valid_files:
		print("nnnn")
		return feedbacks(boardid=int(board_id),comp_selected_list=comp_list,my_designs="All",my_interfaces="All")

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	my_components_id = []
	temp_data = []

	if is_design_layout_lead:
		temp_data = get_all_interfaces_feedbacks(boardid=int(board_id))
	else:
		temp_data = get_my_interfaces_feedbacks(boardid=int(board_id))	

	for i in range(0,len(temp_data[1])):
		my_components_id.append(str(temp_data[1][i][0]))

	print("comp_list: ",comp_list)
	for i in range(len(comp_list)):
		
		comp_id = copy.deepcopy(comp_list[i])
		print("comp_id: ",comp_id)
		print(type(comp_id))
		print("my_components_id: ",my_components_id)

		query = "SELECT * FROM ScheduleTableComponent a WHERE a.BoardID = %s AND a.ComponentID = %s AND a.ScheduleStatusID IN (2,6)"
		val = (board_id,comp_id)
		rs_comp_status = execute_query(query, val)

		is_valid = False
		if rs_comp_status != ():
			if (comp_id in my_components_id) or is_admin:
				is_valid = True


		if is_valid:

			sql = "SELECT * FROM UploadSignOffFilesTemp WHERE BoardID = %s AND ComponentID = %s"
			val = (board_id,comp_id)
			rs_signoff_files_temp = execute_query(sql, val)

			if rs_signoff_files_temp == ():
				sql = "INSERT INTO UploadSignOffFilesTemp SELECT * FROM UploadSignOffFiles WHERE BoardID = %s AND ComponentID = %s"
				val = (board_id,comp_id)
				rs_temp = execute_query(sql, val)

			val = []
			val_latest = []

			#sql = "UPDATE UploadSignOffFiles SET FileName = %s, FileID = %s, ReviewFilenames = %s,Count = %s, WWID = %s, Insert_Time = %s"
			sql = "UPDATE UploadSignOffFilesTemp SET FileName = %s, FileID = %s, ReviewFilenames = %s,Count = %s, WWID = %s, Insert_Time = %s"
			sql_latest = "UPDATE UploadSignoffLatestFiles SET FileName = %s, FileID = %s, ReviewFilenames = %s,Count = %s, WWID = %s, Insert_Time = %s"

			val.append(None)
			val.append(None)
			val.append(None)
			val.append(1)
			val.append(wwid)
			val.append(t)

			val_latest.append(None)
			val_latest.append(None)
			val_latest.append(None)
			val_latest.append(1)
			val_latest.append(wwid)
			val_latest.append(t)

			if request.files['board_file'].filename != '':
				sql += " ,BoardFileID = %s, BoardFileName = %s, BoardCount = %s"
				val.append(board_file_id)
				val.append(board_filename)
				val.append(1)

				sql_latest += " ,BoardFileID = %s, BoardFileName = %s"
				val_latest.append(board_file_id)
				val_latest.append(board_filename)

			if request.files['schematics_file'].filename != '':
				sql += " ,SchematicsFileID = %s, SchematicsName = %s, SchematicsCount = %s"
				val.append(schematics_file_id)
				val.append(schematics_filename)
				val.append(1)

				sql_latest += " ,SchematicsFileID = %s, SchematicsName = %s"
				val_latest.append(schematics_file_id)
				val_latest.append(schematics_filename)

			if request.files['stackup_file'].filename != '':
				sql += " ,StackupFileID = %s, StackupFileName = %s, StackupCount = %s"
				val.append(stackup_file_id)
				val.append(stackup_filename)
				val.append(1)

				sql_latest += " ,StackupFileID = %s, StackupFileName = %s"
				val_latest.append(stackup_file_id)
				val_latest.append(stackup_filename)

			if request.files['lenght_report_file'].filename != '':
				sql += " ,LengthReportFileID = %s, LengthReportFileName = %s, LengthReportCount = %s"
				val.append(lenght_report_file_id)
				val.append(lenght_report_filename)
				val.append(1)

				sql_latest += " ,LengthReportFileID = %s, LengthReportFileName = %s"
				val_latest.append(lenght_report_file_id)
				val_latest.append(lenght_report_filename)

			if request.files['other_files'].filename != '':
				sql += " ,OthersFileID = %s, OthersFileName = %s, OthersCount = %s"
				val.append(other_files_id)
				val.append(other_filesname)
				val.append(1)

				sql_latest += " ,OthersFileID = %s, OthersFileName = %s"
				val_latest.append(other_files_id)
				val_latest.append(other_filesname)

			sql += " WHERE BoardID = %s AND ComponentID = %s"
			val.append(board_id)
			val.append(comp_id)

			sql_latest += " WHERE BoardID = %s"
			val_latest.append(board_id)

			val = tuple(val)
			execute_query(sql,val)

			val_latest = tuple(val_latest)
			execute_query(sql_latest,val_latest)

	return feedbacks(boardid=int(board_id),comp_selected_list=comp_list,my_designs="All",my_interfaces="All")

@app.route("/get_feedbacks_comp",methods = ['POST', 'GET'])
def get_feedbacks_comp():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))

	boardid = copy.deepcopy(data['boardid'])
	my_interfaces = copy.deepcopy(data['my_interfaces'])

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = 'no'
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = 'yes'

	sql = "SELECT * FROM BoardDetails WHERE BoardID = %s AND (DesignLeadWWID = %s OR CADLeadWWID = %s)"	
	val = (boardid,wwid,wwid)		
	rs_design_layout_lead = execute_query(sql,val)

	is_design_layout_lead = False
	if rs_design_layout_lead != ():
		is_design_layout_lead = True

	comp_status_list = []
	comp_list = []
	feedbacks = []
					
	if my_interfaces == "All":
		temp_data = []
		temp_data = get_all_interfaces_feedbacks(boardid=boardid)

	else:
		temp_data = []
		if is_design_layout_lead:
			temp_data = get_all_interfaces_feedbacks(boardid=boardid)
		else:
			temp_data = get_my_interfaces_feedbacks(boardid=boardid)		

	comp_list = temp_data[1]
	#comp_status_list = get_status_list_sorted(data_list=temp_data[0])
	comp_status_list = get_order_status_list(list=temp_data[0])

	final_result = [comp_status_list,comp_list]

	return jsonify(final_result)


@app.route("/get_feedbacks_edit_data",methods = ['POST', 'GET'])
def get_feedbacks_edit_data():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]
	comp_id = data["comp_id"]
	comment_id = data["comment_id"]
	feedbacks_edit_data = []
	edit_part = 0
	is_child = False

	result = {}

	sql = "SELECT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,IFNULL(B1.Comment_Reviewer,''),B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time FROM BoardReview B1 WHERE B1.BoardID = %s AND B1.ComponentID = %s AND B1.CommentID = %s"
	val = (boardid,comp_id,comment_id)
	feedback_rs = execute_query(sql,val)

	if feedback_rs != ():
		feedbacks_edit_data.append(feedback_rs[0][0])
		feedbacks_edit_data.append(feedback_rs[0][1])
		feedbacks_edit_data.append(feedback_rs[0][2])

		# to be filled by electrical 1st part
		feedbacks_edit_data.append(feedback_rs[0][24])	#3
		feedbacks_edit_data.append(feedback_rs[0][3])
		feedbacks_edit_data.append(feedback_rs[0][4])
		feedbacks_edit_data.append(feedback_rs[0][5])

		temp_feedback_summary = copy.deepcopy(feedback_rs[0][6])
		if feedback_rs[0][6] != None:
			temp_feedback_summary = feedback_rs[0][6].replace("---------","")
			if temp_feedback_summary.find("--") != -1:
				feedbacks_edit_data.append(temp_feedback_summary.split("--",1)[0])
			elif temp_feedback_summary.find("<br><br>") != -1:
				feedbacks_edit_data.append(temp_feedback_summary.split("<br><br>",1)[0])
			else:
				feedbacks_edit_data.append(temp_feedback_summary)
		else:
			feedbacks_edit_data.append(temp_feedback_summary)

		feedbacks_edit_data.append(feedback_rs[0][7])	#8
		feedbacks_edit_data.append(feedback_rs[0][23])	#9

		# attachment
		if feedback_rs[0][26] not in ('No File',None,''):
			feedbacks_edit_data.append(True)
		else:
			feedbacks_edit_data.append(False)

		feedbacks_edit_data.append(feedback_rs[0][8])	#11

		temp_comment = copy.deepcopy(feedback_rs[0][9])
		if feedback_rs[0][9] != None:
			temp_comment = feedback_rs[0][9].replace("---------","")
			if temp_comment.find("--") != -1:
				feedbacks_edit_data.append(temp_comment.split("--",1)[0])
			elif temp_comment.find("<br><br>") != -1:
				feedbacks_edit_data.append(temp_comment.split("<br><br>",1)[0])
			else:
				feedbacks_edit_data.append(temp_comment)
		else:
			feedbacks_edit_data.append(temp_comment)

		# attachment
		if feedback_rs[0][27] not in ('No File',None,''):	#13
			feedbacks_edit_data.append(True)
		else:
			feedbacks_edit_data.append(False)


		# to be filled by electrical 2nd part
		feedbacks_edit_data.append(feedback_rs[0][19])	#14

		temp_comment = copy.deepcopy(feedback_rs[0][20])
		if feedback_rs[0][20] != None:
			temp_comment = feedback_rs[0][20].replace("---------","")
			if temp_comment.find("--") != -1:
				feedbacks_edit_data.append(temp_comment.split("--",1)[0])
			elif temp_comment.find("<br><br>") != -1:
				feedbacks_edit_data.append(temp_comment.split("<br><br>",1)[0])
			else:
				feedbacks_edit_data.append(temp_comment)
		else:
			feedbacks_edit_data.append(temp_comment)				

		feedbacks_edit_data.append(feedback_rs[0][21])

		# to set edit part based on submitted indicator
		if (feedback_rs[0][18] == "yes"):
			edit_part = 3

		elif (feedback_rs[0][16] == "yes"):
			edit_part = 2

		elif (feedback_rs[0][14] == "yes"):
			edit_part = 1

		feedbacks_edit_data.append(edit_part)	#17
		
		if feedback_rs[0][12] != 0:
			is_child = True

		if feedback_rs[0][26] not in ('No File',None,''):	#18
			feedbacks_edit_data.append('Download Attachment\nFile Name: '+str(feedback_rs[0][26]))
		else:
			feedbacks_edit_data.append("")

		# attachment
		if feedback_rs[0][27] not in ('No File',None,''):	#19
			feedbacks_edit_data.append('Download Attachment\nFile Name: '+str(feedback_rs[0][27]))
		else:
			feedbacks_edit_data.append("")

	result["feedbacks_edit_data"] = json.dumps(feedbacks_edit_data)
	result["edit_part"] = json.dumps(edit_part)
	result["is_child"] = json.dumps(is_child)

	return jsonify(result)


@app.route("/get_design_feedbacks_edit_data",methods = ['POST', 'GET'])
def get_design_feedbacks_edit_data():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]
	comp_id = data["comp_id"]
	comment_id = data["comment_id"]
	feedbacks_edit_data = []
	edit_part = 0
	is_child = False

	result = {}

	sql = "SELECT B1.BoardID,B1.ComponentID,B1.CommentID,B1.DesignDocument,B1.SignalName,B1.AreaOfIssue,B1.FeedbackSummary,B1.RiskLevel,B1.ImplementationStatus,B1.Comment,B1.ReviewerFeedbackGiven,B1.DesignerFeedbackGiven,B1.ParentCommentID,B1.Saved,B1.Submitted,B1.Saved_Designer,B1.Submitted_Designer,B1.SignedOff_Reviewer2,B1.Submitted_Reviewer2,B1.IssueStatus,IFNULL(B1.Comment_Reviewer,''),B1.RiskLevelSignOff,B1.Saved_Reviewer2,B1.ReferenceNumber,B1.BoardFileName,B1.HasChild,B1.ReviewerFileName,B1.DesignerFileName,B1.ActualParentID,B1.WWIDreviewer,B1.WWIDdesigner,B1.Submit_Time FROM BoardReview B1 WHERE B1.BoardID = %s AND B1.ComponentID = %s AND B1.CommentID = %s"
	val = (boardid,comp_id,comment_id)
	feedback_rs = execute_query(sql,val)

	if feedback_rs != ():
		feedbacks_edit_data.append(feedback_rs[0][0])
		feedbacks_edit_data.append(feedback_rs[0][1])
		feedbacks_edit_data.append(feedback_rs[0][2])

		# to be filled by electrical 1st part
		feedbacks_edit_data.append(feedback_rs[0][24])	#3
		feedbacks_edit_data.append(feedback_rs[0][3])
		feedbacks_edit_data.append(feedback_rs[0][4])
		feedbacks_edit_data.append(feedback_rs[0][5])

		temp_feedback_summary = copy.deepcopy(feedback_rs[0][6])
		if feedback_rs[0][6] != None:
			temp_feedback_summary = feedback_rs[0][6].replace("---------","")
			if temp_feedback_summary.find("--") != -1:
				feedbacks_edit_data.append(temp_feedback_summary.split("--",1)[0])
			else:
				feedbacks_edit_data.append(temp_feedback_summary)
		else:
			feedbacks_edit_data.append(temp_feedback_summary)

		feedbacks_edit_data.append(feedback_rs[0][7])	#8
		feedbacks_edit_data.append(feedback_rs[0][23])	#9

		# attachment
		if feedback_rs[0][26] not in ('No File',None,''):
			feedbacks_edit_data.append(True)
		else:
			feedbacks_edit_data.append(False)

		# to be filled by design owner
		feedbacks_edit_data.append(feedback_rs[0][8])	#11

		temp_comment = copy.deepcopy(feedback_rs[0][9])
		if feedback_rs[0][9] != None:
			temp_comment = feedback_rs[0][9].replace("---------","")
			if temp_comment.find("--") != -1:
				feedbacks_edit_data.append(temp_comment.split("--",1)[0])
			else:
				feedbacks_edit_data.append(temp_comment)
		else:
			feedbacks_edit_data.append(temp_comment)

		# attachment
		if feedback_rs[0][27] not in ('No File',None,''):	#13
			feedbacks_edit_data.append(True)
		else:
			feedbacks_edit_data.append(False)


		# to be filled by electrical 2nd part
		feedbacks_edit_data.append(feedback_rs[0][19])	#14

		temp_comment = copy.deepcopy(feedback_rs[0][20])
		if feedback_rs[0][20] != None:
			temp_comment = feedback_rs[0][20].replace("---------","")
			if temp_comment.find("--") != -1:
				feedbacks_edit_data.append(temp_comment.split("--",1)[0])
			else:
				feedbacks_edit_data.append(temp_comment)
		else:
			feedbacks_edit_data.append(temp_comment)				

		feedbacks_edit_data.append(feedback_rs[0][21])

		# to set edit part based on submitted indicator
		if (feedback_rs[0][18] == "yes"):
			edit_part = 3

		elif (feedback_rs[0][16] == "yes"):
			edit_part = 2

		elif (feedback_rs[0][14] == "yes"):
			edit_part = 1

		feedbacks_edit_data.append(edit_part)	#18
		
		if feedback_rs[0][12] != 0:
			is_child = True

		if feedback_rs[0][26] not in ('No File',None,''):	#18
			feedbacks_edit_data.append('Download Attachment\nFile Name: '+str(feedback_rs[0][26]))
		else:
			feedbacks_edit_data.append("")

		# attachment
		if feedback_rs[0][27] not in ('No File',None,''):	#19
			feedbacks_edit_data.append('Download Attachment\nFile Name: '+str(feedback_rs[0][27]))
		else:
			feedbacks_edit_data.append("")

	result["feedbacks_edit_data"] = json.dumps(feedbacks_edit_data)
	result["edit_part"] = json.dumps(edit_part)
	result["is_child"] = json.dumps(is_child)
	#result["comp_status_list"] = json.dumps("")

	return jsonify(result)

@app.route("/get_download_files_details",methods = ['POST', 'GET'])
def get_download_files_details():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]
	comp_list = data["comp_list"]
	btn_value = data["btn"]

	result = {}

	sql_fileid = ""
	sql_filename = ""

	if btn_value == "0":
		sql_fileid = "FileID"
		sql_filename = "ReviewFilenames"

	if btn_value == "1":
		sql_fileid = "BoardFileID"
		sql_filename = "BoardFileName"

	if btn_value == "2":
		sql_fileid = "SchematicsFileID"
		sql_filename = "SchematicsName"

	if btn_value == "3":
		sql_fileid = "StackupFileID"
		sql_filename = "StackupFileName"

	if btn_value == "4":
		sql_fileid = "LengthReportFileID"
		sql_filename = "LengthReportFileName"

	if btn_value == "5":
		sql_fileid = "OthersFileID"
		sql_filename = "OthersFileName"

	sql = "SELECT "+sql_fileid+","+sql_filename+" FROM UploadSignOffFiles WHERE BoardID = %s AND ComponentID IN %s GROUP BY "+sql_fileid+" HAVING "+sql_fileid+" > 0"
	val = (boardid,comp_list)

	rs_file_details = execute_query(sql,val)

	file_comp_level_details = []
	if rs_file_details != ():

		for i in range(0,len(rs_file_details)):

			temp_comp_details = []

			sql="SELECT a.UploadTime,b.UserName FROM UploadFileStorageMisc a LEFT JOIN HomeTable b ON a.UploadedBy = b.WWID WHERE a.FileID = %s AND a.BoardID = %s"
			val=(rs_file_details[i][0],boardid)
			upload_details_rs=execute_query(sql,val)

			upload_time = ''
			upload_by = ''

			if upload_details_rs != ():
				upload_time = str(get_work_week_fun_with_year(upload_details_rs[0][0]))
				upload_by = upload_details_rs[0][1]

			sql = "SELECT a.ComponentID,b.ComponentName FROM UploadSignOffFiles a LEFT JOIN ComponentType b ON a.ComponentID = b.ComponentID WHERE a.BoardID = %s AND a.ComponentID IN %s AND a."
			sql += sql_fileid+"=%s ORDER BY b.ComponentName"
			val = (boardid,comp_list,rs_file_details[i][0])
			rs_comp_level_details = execute_query(sql,val)

			for j in range(0,len(rs_comp_level_details)):
				temp_comp_details.append([rs_comp_level_details[j][0],rs_comp_level_details[j][1]])

			file_comp_level_details.append([temp_comp_details,rs_file_details[i][1].replace(';', '<br>'),rs_file_details[i][0],upload_time,upload_by])

	result["file_comp_level_details"] = json.dumps(file_comp_level_details)

	return jsonify(result)

@app.route("/get_download_files_details_all_zip",methods = ['POST', 'GET'])
def get_download_files_details_all_zip():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]
	comp_list = data["comp_list"]

	result = {}
	
	sql = "SELECT BoardFileID,SchematicsFileID,StackupFileID,LengthReportFileID,OthersFileID FROM UploadSignOffFiles WHERE BoardID = %s AND ComponentID IN %s GROUP BY BoardFileID,SchematicsFileID,StackupFileID,LengthReportFileID,OthersFileID HAVING BoardFileID > 0"
	val = (boardid,comp_list)

	rs_file_details = execute_query(sql,val)

	file_comp_level_details = []
	if rs_file_details != ():

		for i in range(0,len(rs_file_details)):

			temp_comp_details = []

			sql="SELECT a.UploadTime,b.UserName FROM UploadFileStorageMisc a LEFT JOIN HomeTable b ON a.UploadedBy = b.WWID WHERE a.BoardID = %s AND a.FileID IN %s ORDER BY a.UploadTime DESC LIMIT 1"
			val=(boardid,[rs_file_details[i][0],rs_file_details[i][1],rs_file_details[i][2],rs_file_details[i][3],rs_file_details[i][4]])
			upload_details_rs=execute_query(sql,val)

			upload_time = ''
			upload_by = ''

			if upload_details_rs != ():
				upload_time = str(get_work_week_fun_with_year(upload_details_rs[0][0]))
				upload_by = upload_details_rs[0][1]

			val = [boardid,comp_list]
			sql = "SELECT a.ComponentID,b.ComponentName FROM UploadSignOffFiles a LEFT JOIN ComponentType b ON a.ComponentID = b.ComponentID WHERE a.BoardID = %s AND a.ComponentID IN %s"
			if rs_file_details[i][0] is None:
				sql += " AND a.BoardFileID IS NULL"
			else:
				sql += " AND a.BoardFileID = %s"
				val.append(rs_file_details[i][0])

			if rs_file_details[i][1] is None:
				sql += " AND a.SchematicsFileID IS NULL"
			else:
				sql += " AND a.SchematicsFileID = %s"
				val.append(rs_file_details[i][1])

			if rs_file_details[i][2] is None:
				sql += " AND a.StackupFileID IS NULL"
			else:
				sql += " AND a.StackupFileID = %s"
				val.append(rs_file_details[i][2])

			if rs_file_details[i][3] is None:
				sql += " AND a.LengthReportFileID IS NULL"
			else:
				sql += " AND a.LengthReportFileID = %s"
				val.append(rs_file_details[i][3])

			if rs_file_details[i][4] is None:
				sql += " AND a.OthersFileID IS NULL"
			else:
				sql += " AND a.OthersFileID = %s"
				val.append(rs_file_details[i][4])

			sql += " ORDER BY b.ComponentName"
			val = tuple(val)
			rs_comp_level_details = execute_query(sql,val)

			for j in range(0,len(rs_comp_level_details)):
				temp_comp_details.append([rs_comp_level_details[j][0],rs_comp_level_details[j][1]])

			file_comp_level_details.append([temp_comp_details,'',[rs_file_details[i][0],rs_file_details[i][1],rs_file_details[i][2],rs_file_details[i][3],rs_file_details[i][4]],upload_time,upload_by])

	result["file_comp_level_details"] = json.dumps(file_comp_level_details)

	return jsonify(result)

@app.route("/check_for_design_files",methods = ['POST', 'GET'])
def check_for_design_files():

	wwid=session.get('wwid')
	username = session.get('username')

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]
	comp_list = data["comp_list"]

	result = {}
	available_comp_id_list = []

	err_msg = ''

	sql="SELECT BoardID,FileName,FileID,ReviewFilenames,WWID,Insert_Time,BoardFileID,BoardFileName,SchematicsFileID,SchematicsName,StackupFileID,StackupFileName,LengthReportFileID,LengthReportFileName,OthersFileID,OthersFileName,Count FROM UploadDesignFiles WHERE BoardID = %s"
	val = (boardid,)
	rs_design_files = execute_query(sql,val)

	if rs_design_files == ():
		
		err_msg = 'Please upload design files'
		result["err_msg"] = err_msg

		return jsonify(result)		

	else:
		sql="SELECT BoardID,ComponentID FROM UploadSignOffFiles WHERE BoardID = %s"
		val = (boardid,)
		rs_available_signoff_files = execute_query(sql,val)

		if rs_available_signoff_files != ():

			for i in range(0,len(rs_available_signoff_files)):

				available_comp_id_list.append(str(rs_available_signoff_files[i][1]))

		for comp_id in comp_list:

			if comp_id not in available_comp_id_list:

				sql = "INSERT INTO UploadSignOffFiles VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
				val = (boardid,comp_id,rs_design_files[0][1],rs_design_files[0][2],rs_design_files[0][3],0,wwid,t,rs_design_files[0][6],rs_design_files[0][7],0,rs_design_files[0][8],rs_design_files[0][9],0,rs_design_files[0][10],rs_design_files[0][11],0,rs_design_files[0][12],rs_design_files[0][13],0,rs_design_files[0][14],rs_design_files[0][15],0)
				execute_query(sql,val)		

			# only yet to Kickstart status of Interfaces only allowed to update base design files
			query = "SELECT ScheduleStatusID FROM ScheduleTableComponent WHERE BoardID = %s AND ComponentID = %s"
			val=(boardid,comp_id)
			status_check=execute_query(query,val)

			status_check_valid = False

			if status_check == ():
				status_check_valid = True
			else:
				if status_check[0][0] in [3,'3']:
					status_check_valid = True

			if status_check_valid:
				sql = "INSERT INTO ScheduleTableComponent VALUES (%s,%s,%s) on DUPLICATE KEY UPDATE ScheduleStatusID = ScheduleStatusID"
				val=(boardid,comp_id,3)
				execute_query(sql,val)

				t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

				sql = "INSERT INTO UploadSignOffFiles VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s) ON DUPLICATE KEY UPDATE FileName = %s, FileID = %s, ReviewFilenames = %s, WWID = %s, Insert_Time = %s, BoardFileID = %s, BoardFileName = %s, SchematicsFileID = %s, SchematicsName = %s, StackupFileID = %s, StackupFileName = %s, LengthReportFileID = %s, LengthReportFileName = %s, OthersFileID = %s, OthersFileName = %s"
				#val = (board_id,comp_id,filename,file_id,reviewfiles,0,wwid,t,board_file_id,board_filename,0,schematics_file_id,schematics_filename,0,stackup_file_id,stackup_filename,0,lenght_report_file_id,lenght_report_filename,0,other_files_id,other_filesname,0,filename,file_id,reviewfiles,wwid,t,board_file_id,board_filename,schematics_file_id,schematics_filename,stackup_file_id,stackup_filename,lenght_report_file_id,lenght_report_filename,other_files_id,other_filesname)
				val = (boardid,comp_id,rs_design_files[0][1],rs_design_files[0][2],rs_design_files[0][3],0,wwid,t,rs_design_files[0][6],rs_design_files[0][7],0,rs_design_files[0][8],rs_design_files[0][9],0,rs_design_files[0][10],rs_design_files[0][11],0,rs_design_files[0][12],rs_design_files[0][13],0,rs_design_files[0][14],rs_design_files[0][15],0,rs_design_files[0][1],rs_design_files[0][2],rs_design_files[0][3],wwid,t,rs_design_files[0][6],rs_design_files[0][7],rs_design_files[0][8],rs_design_files[0][9],rs_design_files[0][10],rs_design_files[0][11],rs_design_files[0][12],rs_design_files[0][13],rs_design_files[0][14],rs_design_files[0][15])
				execute_query(sql,val)

				sql = "INSERT INTO UploadSignoffLatestFiles VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s) ON DUPLICATE KEY UPDATE FileName = %s, FileID = %s, ReviewFilenames = %s, WWID = %s, Insert_Time = %s, BoardFileID = %s, BoardFileName = %s, SchematicsFileID = %s, SchematicsName = %s, StackupFileID = %s, StackupFileName = %s, LengthReportFileID = %s, LengthReportFileName = %s, OthersFileID = %s, OthersFileName = %s"
				#val = (board_id,filename,file_id,reviewfiles,wwid,t,board_file_id,board_filename,schematics_file_id,schematics_filename,stackup_file_id,stackup_filename,lenght_report_file_id,lenght_report_filename,other_files_id,other_filesname,0,filename,file_id,reviewfiles,wwid,t,board_file_id,board_filename,schematics_file_id,schematics_filename,stackup_file_id,stackup_filename,lenght_report_file_id,lenght_report_filename,other_files_id,other_filesname)
				val = (boardid,rs_design_files[0][1],rs_design_files[0][2],rs_design_files[0][3],wwid,t,rs_design_files[0][6],rs_design_files[0][7],rs_design_files[0][8],rs_design_files[0][9],rs_design_files[0][10],rs_design_files[0][11],rs_design_files[0][12],rs_design_files[0][13],rs_design_files[0][14],rs_design_files[0][15],0,rs_design_files[0][1],rs_design_files[0][2],rs_design_files[0][3],wwid,t,rs_design_files[0][6],rs_design_files[0][7],rs_design_files[0][8],rs_design_files[0][9],rs_design_files[0][10],rs_design_files[0][11],rs_design_files[0][12],rs_design_files[0][13],rs_design_files[0][14],rs_design_files[0][15])
				execute_query(sql,val)


	sql="SELECT BoardID,ComponentID FROM UploadSignOffFiles WHERE BoardID = %s AND (BoardFileID IS NOT NULL) AND (SchematicsFileID IS NOT NULL) AND (StackupFileID IS NOT NULL)"
	val = (boardid,)
	rs_signoff_files = execute_query(sql,val)

	for comp_id in comp_list:

		is_valid_flag = False
		for i in range(0,len(rs_signoff_files)):

			if (str(comp_id) == str(rs_signoff_files[i][1])):

				is_valid_flag = True
				break
		
		if not is_valid_flag:
			sql="SELECT ComponentName FROM ComponentType WHERE ComponentID = %s"
			val = (comp_id,)
			rs_comp_name=execute_query(sql,val)

			if rs_comp_name != ():
				err_msg = 'Please upload design files for '+rs_comp_name[0][0]

			result["err_msg"] = err_msg
			return jsonify(result)


	result["err_msg"] = err_msg

	return jsonify(result)

@app.route('/logs.html', methods=['POST', 'GET'])
def logs():

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'logs'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	name =  session.get('username')
	wwid =  session.get('wwid')

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	page = request.args.get("page")

	if not is_admin:
		return render('error_custom.html',error='You do not have access to this link.',username=username,user_role_name=user_role_name,region_name=region_name)

	if page is None:
		page=0

	page=int(page)
	
	sql="select RoleID from HomeTable where WWID=%s "
	val = (wwid,)
	role=execute_query(sql,val)[0][0]
	if (role == 14):
		mgt_access=True
	else:
		mgt_access=False

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	perpage=100
	start_rec = int(page*perpage)

	sql = "SELECT l.LogID,l.Category,l.BoardID,l.RequestID,h.UserName,l.LogTime,l.Comments FROM LogTable l LEFT JOIN HomeTable h on h.WWID = l.WWID ORDER BY l.LogID DESC"
	sql += " LIMIT %s,%s"
	val=(start_rec,perpage)
	result = execute_query(sql,val)

	logs = []
	for i in range(0,len(result)):
		temp = []
		#temp.append(result[i][5])
		temp.append(get_work_week_date_fmt(result[i][5])+' - '+str(result[i][5].strftime("%H:%M:%S")))
		temp.append(result[i][1])
		temp.append(result[i][4])
		temp.append(result[i][6])

		logs.append(temp)

	if page < 1:
		prev_page = 0
	else:
		prev_page = page-1

	next_page = page+1
	if len(result) < perpage:
		next_page = 0

	return render("logs.html",is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,mgt_access=mgt_access,is_admin=is_admin,name=name,logs=logs,page=page,prev_page=prev_page,next_page=next_page,start_rec=start_rec,username=username,user_role_name=user_role_name,region_name=region_name)

@app.route("/temp_file_upload",methods = ['POST', 'GET'])
def temp_file_upload():
	name = session.get('username')
	if request.method == 'POST':

		comment_id=request.form.get('comment_id')

		try:
			electrical_file = request.files["elec_file"]
		except:
			electrical_file = None

		try:
			design_file = request.files["design_file"]
		except:
			design_file = None

		# files upload
		if electrical_file is not None:
			if electrical_file.filename != '':
				file=electrical_file.read()
				filename=electrical_file.filename
				fname = filename

				sql = "INSERT INTO FileStorage (CommentID,ReviewerFilename,ReviewerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE ReviewerFilename = %s, ReviewerFile = %s"
				val = (comment_id,fname,file,fname,file)
				execute_query(sql,val)

		if design_file is not None:
			if design_file.filename != '':
				file=design_file.read()
				filename=design_file.filename
				fname = filename

				sql = "INSERT INTO FileStorage (CommentID,DesignerFilename,DesignerFile) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE DesignerFilename = %s, DesignerFile = %s"
				val = (comment_id,fname,file,fname,file)
				execute_query(sql,val)

	return render("temp_file_upload.html",name = name)

@app.route("/about.html")
def about():
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	return render("about.html",username=username,user_role_name=user_role_name,region_name=region_name)

@app.route("/change_user.html")
def change_user():
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	return render("user_set.html",username=username,user_role_name=user_role_name,region_name=region_name)

#Method called when the request access page is loaded. It gets the details and stores it in the DB.
@app.route('/user_set',methods = ['POST', 'GET'])
def user_set():
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	if str(session.get('sso_loggedin_wwid')) != "11898148":
		if is_prod:
			return render('error_custom.html',error='Not Authorized',username=username,user_role_name=user_role_name,region_name=region_name)


	wwid_form=request.form.get('wwid_form')

	sql="SELECT a.WWID,a.UserName,a.EmailID,b.RoleName,a.IsActive FROM HomeTable a LEFT JOIN RoleTable b ON a.RoleID=b.RoleID WHERE a.WWID=%s"
	val=(wwid_form,)
	result=execute_query(sql,val)

	if result != ():
		session['username'] = result[0][1]
		session['wwid']= str(result[0][0])
		session['email'] = result[0][2]
		session['is_admin'] = False
		session['user_role_name'] = result[0][3]
		session['is_inactive_user'] = False
		session['inactive_user_msg'] = ""

		if result[0][4] == 0:
			session['is_inactive_user'] = True
			session['inactive_user_msg'] = "Your "+str(result[0][3])+" role access is being revoked. Please re-apply to get access."

		sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
		val = (wwid_form,)
		has_admin_access = execute_query(sql,val)

		if has_admin_access != ():
			if(has_admin_access[0][0] == 'yes'):
				session['is_admin'] = True

	else:
		return render('error_custom.html',error='User not available in tool',username=username,user_role_name=user_role_name,region_name=region_name)

	return redirect(url_for('index', _external=True))

#Method called when the request access page is loaded. It gets the details and stores it in the DB.
@app.route('/role_change',methods = ['POST', 'GET'])
def role_change():

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'role_change'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	wwid =  session.get('wwid')
	name = session.get('username')

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	email_list = []

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	sql = "SELECT a.RoleID,b.RoleName,a.EmailID FROM HomeTable a LEFT JOIN RoleTable b ON a.RoleID=b.RoleID WHERE a.WWID = %s"
	val = (wwid,)
	role_name_rs = execute_query(sql,val)

	if request.method == 'POST':

		role=request.form.get('role')
		reason=request.form.get('reason')

		sql = "INSERT INTO RoleChangeRequest (RequestID,WWID,CurrentRoleID,RequestedRoleID,Reason,StateID,Request_Time) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = (0,wwid,role_name_rs[0][0],role,reason,1,t)
		execute_query(sql,val)

		sql = "SELECT LAST_INSERT_ID()" #Returns request ID
		request_id = str(execute_query_sql(sql)[0][0])

		# log table
		try:
			log_notes = 'User has raised Role Change request'
			log_wwid = session.get('wwid')
			t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
			sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
			val = ('Role Change Request',0,request_id[0][0],log_wwid,t,log_notes)
			execute_query(sql,val)
		except:
			if is_logging:
				logging.exception('')
			print("log error.")

		sql = "SELECT RoleName FROM RoleTable WHERE RoleID = %s"
		val = (role,)
		role_name_req_rs = execute_query(sql,val)

		role_name_requested = ''
		if role_name_req_rs != ():
			role_name_requested = role_name_req_rs[0][0]

		subject="[Request ID :"+str(request_id)+"] Role Change Request Submitted"
		message='''Hi,<br><br> ERAM - Role Change Request has been Submitted by <b>'''+str(name)+'''</b><br><br>
		<b>Request ID: </b>'''+request_id+'''<br>
		<b>User Name: </b>'''+name+'''<br>
		<b>Current Role: </b>'''+role_name_rs[0][1]+'''<br>
		<b>Requested Role: </b><u>'''+role_name_requested+'''</u><br>
		<b>Comments: </b>'''+str(reason)+'''<br><br>

		Thanks, <br>ERAM.'''
		
		# requester email id
		email_list.append(role_name_rs[0][2])

		sql = "SELECT EmailID FROM HomeTable WHERE AdminAccess = %s"
		val = ("yes",)
		admin_rs = execute_query(sql,val)

		for i in range(len(admin_rs)):
			email_list.append(admin_rs[i][0])

		email_list = sorted(set(email_list), reverse=True)

		for m in email_list:
			send_mail(m,subject,message,email_list)

		return render('custom_message.html',username=username,user_role_name=user_role_name,region_name=region_name,color="green",message='Request has been submitted successfully.')

	else:

		sql = "SELECT RequestID FROM RoleChangeRequest WHERE WWID = %s AND StateID = %s"
		val = (wwid,1)
		check_request = execute_query(sql,val)

		if check_request != ():
			return render('error_custom.html',error='Request has been raised already. Request ID: '+str(check_request[0][0]),username=username,user_role_name=user_role_name,region_name=region_name)

		current_role = ''
		if role_name_rs != ():
			current_role = role_name_rs[0][1]

		sql = "SELECT RoleID,RoleName FROM RoleTable WHERE RoleID <> %s ORDER BY RoleName"
		val = (role_name_rs[0][0],)
		roles = execute_query(sql,val)
		roles_list=[]
		for i in roles:
			roles_list.append([i[0],i[1]])

		return render('role_change.html',is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,is_admin=is_admin,role=roles_list,current_role=current_role,username=username,user_role_name=user_role_name,region_name=region_name)

#Method called when the request access page is loaded. It gets the details and stores it in the DB.
@app.route('/revoke_access',methods = ['POST', 'GET'])
def revoke_access():

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'role_change'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	wwid =  session.get('wwid')
	name = session.get('username')

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	email_list = []

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	sql = "SELECT a.RoleID,b.RoleName,a.EmailID FROM HomeTable a LEFT JOIN RoleTable b ON a.RoleID=b.RoleID WHERE a.WWID = %s"
	val = (wwid,)
	role_name_rs = execute_query(sql,val)

	if request.method == 'POST':

		reason=request.form.get('reason')

		sql = "INSERT INTO RevokeAccess (RequestID,WWID,RoleID,Comments,StateID,Request_Time) VALUES (%s,%s,%s,%s,%s,%s)"
		val = (0,wwid,role_name_rs[0][0],reason,2,t)
		execute_query(sql,val)

		sql = "SELECT LAST_INSERT_ID()" #Returns request ID
		request_id = str(execute_query_sql(sql)[0][0])

		sql = "UPDATE HomeTable SET IsActive = %s WHERE WWID = %s"
		val = (0,wwid)
		execute_query(sql,val)

		# log table
		try:
			log_notes = 'User has raised Revoke Access request'
			log_wwid = session.get('wwid')
			t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
			sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
			val = ('Revoke Access Request',0,request_id[0][0],log_wwid,t,log_notes)
			execute_query(sql,val)
		except:
			if is_logging:
				logging.exception('')
			print("log error.")

		subject="[Request ID :"+str(request_id)+"] Access Revoked"
		message='''Hi,<br><br> ERAM - Board Tool Access has been Revoked by <b>'''+str(name)+'''</b> for below user,<br><br>
		<b>Request ID: </b>'''+request_id+'''<br>
		<b>User Name: </b>'''+name+'''<br>
		<b>Role: </b>'''+role_name_rs[0][1]+'''<br>
		<b>Comments: </b>'''+str(reason)+'''<br><br>

		Thanks, <br>ERAM.'''
		
		# requester email id
		email_list.append(role_name_rs[0][2])

		sql = "SELECT EmailID FROM HomeTable WHERE AdminAccess = %s"
		val = ("yes",)
		admin_rs = execute_query(sql,val)

		for i in range(len(admin_rs)):
			email_list.append(admin_rs[i][0])

		email_list = sorted(set(email_list), reverse=True)

		for m in email_list:
			send_mail(m,subject,message,email_list)

		# clear session data to disable user
		session.clear()

		return render('custom_message.html',username=username,user_role_name=user_role_name,region_name=region_name,color="green",message='Access has been revoked successfully.')

	return render('revoke_access.html',is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,is_admin=is_admin,username=username,user_role_name=user_role_name,region_name=region_name)

@app.route('/role_change_accept',methods = ['POST', 'GET'])
def role_change_accept():
	approveid = request.form.get('approveid')

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	# log table
	try:
		log_notes = 'Admin has Accepted Role Change access request for Request ID: '+str(approveid)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
		val = ('Role Change Request',0,approveid,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")

	sql = "SELECT a.RequestID,a.WWID,a.RequestedRoleID,b.EmailID,REPLACE(b.UserName,',',' ') FROM RoleChangeRequest a,HomeTable b WHERE a.RequestID = %s AND a.WWID=b.WWID"
	val = (approveid,)
	approved_details = execute_query(sql,val)

	if approved_details == ():
		return render('error_custom.html',error='Error Occured',username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "UPDATE HomeTable SET RoleID = %s WHERE WWID = %s"
	val=(approved_details[0][2],approved_details[0][1])
	execute_query(sql,val)

	sql = "UPDATE RequestAccess SET RoleID = %s WHERE WWID = %s"
	val=(approved_details[0][2],approved_details[0][1])
	execute_query(sql,val)

	sql = "UPDATE RoleChangeRequest SET StateID = %s WHERE RequestID = %s"
	val=(2,approveid)
	execute_query(sql,val)

	message = '''Hi '''+str(approved_details[0][4])+''',<br><br>
		Your request for Role Change to ERAM tool has been accepted. <br><br>Please visit https://eram.apps1-fm-int.icloud.intel.com/ to proceed further. <br><br>Thanks,<br>ERAM.'''
	subject="[Request ID: "+str(approveid)+"] Request for Role Change has been approved."

	email_list = []
	
	# requester email id
	email_list.append(approved_details[0][3])

	sql = "SELECT EmailID FROM HomeTable WHERE AdminAccess = %s"
	val = ("yes",)
	admin_rs = execute_query(sql,val)

	for i in range(len(admin_rs)):
		email_list.append(admin_rs[i][0])

	email_list = sorted(set(email_list), reverse=True)
	for m in email_list:
		send_mail(m,subject,message,email_list)			

	return redirect(url_for('review_request', _external=True))

@app.route('/role_change_reject',methods = ['POST', 'GET'])
def role_change_reject():
	rejectid = request.form.get('rejectid')
	rejectreason=request.form.get("reject_reason")
	user_wwid = session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')


	# log table
	try:
		log_notes = 'Admin has Rejected Role Change access request for Request ID: '+str(rejectid)+'<br>Comments: '+str(rejectreason)
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
		val = ('Role Change Request',0,rejectid,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")
	
	sql = "SELECT a.RequestID,a.WWID,a.RequestedRoleID,b.EmailID,REPLACE(b.UserName,',',' ') FROM RoleChangeRequest a,HomeTable b WHERE a.RequestID = %s AND a.WWID=b.WWID"
	val = (rejectid,)
	approved_details = execute_query(sql,val)

	if approved_details == ():
		return render('error_custom.html',error='Error Occured',username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "UPDATE RoleChangeRequest SET StateID = %s WHERE RequestID = %s"
	val=(3,rejectid)
	execute_query(sql,val)

	message = '''Hi '''+str(approved_details[0][4])+''',<br><br>
		Your request for Role Change to ERAM tool has been Rejected.<br><br>Comments: '''+str(rejectreason)+'''<br><br> Please visit https://eram.apps1-fm-int.icloud.intel.com/ to proceed further. <br><br>Thanks,<br>ERAM.'''
	subject="[Request ID: "+str(rejectid)+"] Request for Role Change has been Rejected."

	email_list = []
	
	# requester email id
	email_list.append(approved_details[0][3])

	sql = "SELECT EmailID FROM HomeTable WHERE AdminAccess = %s"
	val = ("yes",)
	admin_rs = execute_query(sql,val)

	for i in range(len(admin_rs)):
		email_list.append(admin_rs[i][0])

	email_list = sorted(set(email_list), reverse=True)
	for m in email_list:
		send_mail(m,subject,message,email_list)

	return redirect(url_for('review_request', _external=True))


@app.route('/design_owners', methods=['POST', 'GET'])
def design_owners(boardid=0,platform_id=0,sku_id=0,memory_type_id=0,design_type_id=0):

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'design_owners'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	wwid =  session.get('wwid')
	username =  session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	board_name = ''

	'''
	boardid = 0
	platform_id = 0
	sku_id = 0
	memory_type_id = 0
	design_type_id = 0
	'''

	data = []
	error_message = ''
	edit_access = False

	head_title = ""

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	is_admin = False
	if has_admin_access == "yes":
		is_admin = True
		edit_access = True

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	# only admin, design lead & manager can have edit access
	if(role == 3 or role == 5):
		edit_access = True

	if (role in [3,5,6,7,8,9]) or is_admin:
		pass
	else:
		return render('error_custom.html',error='You do not have access to this page. Please contact admin.',username=username,user_role_name=user_role_name,region_name=region_name)

	#sql="SELECT SKUID,SKUName FROM SUK ORDER BY SKUName"
	sql="SELECT SKUID,SKUName FROM SUK WHERE SKUID <> 27 ORDER BY SKUName"
	sku=execute_query_sql(sql)

	sql="SELECT PlatformID,PlatformName FROM Platform ORDER BY PlatformName"
	plat=execute_query_sql(sql)

	sql="SELECT MemTypeID,MemTypeName FROM MemType ORDER BY MemTypeName"
	mem=execute_query_sql(sql)

	sql="SELECT DesignTypeID,DesignTypeName FROM DesignType ORDER BY DesignTypeName"
	des=execute_query_sql(sql)

	sql = "SELECT DISTINCT WWID,UserName,RoleID FROM HomeTable WHERE IsActive = 1 AND RoleID IN (3,5,6,7,8,9) ORDER BY RoleID,UserName"
	user_list = execute_query_sql(sql)

	sql = "SELECT DISTINCT RoleID,RoleName FROM RoleTable WHERE RoleID IN (3,5,6,7,8,9) ORDER BY RoleName"
	role_list = execute_query_sql(sql)

	temp_data = []
	design_status_list = []
	design_list = []

	edit_mode = False
	is_imported = False

	temp_data = get_all_designs_owners()
	#design_status_list = get_status_list_sorted(data_list=temp_data[0])
	design_status_list = get_order_status_list(list=temp_data[0])
	design_list = temp_data[1]

	if request.method == 'POST':

		if (platform_id is None) or (str(platform_id) == str(0)):
			boardid = request.form.get("boardid")
			platform_id = request.form.get("platform_id")
			sku_id = request.form.get("sku_id")
			memory_type_id = request.form.get("memory_type_id")
			design_type_id = request.form.get("design_type_id")

			sql = "SELECT * FROM (SELECT a.PlatformName FROM Platform a WHERE a.PlatformID = %s) AS platform, (SELECT a.SKUName FROM SUK a WHERE a.SKUID = %s) AS sku, (SELECT a.MemTypeName FROM MemType a WHERE a.MemTypeID = %s) AS mem_type, (SELECT a.DesignTypeName FROM DesignType a WHERE a.DesignTypeID = %s) AS design_type"
			val = (platform_id,sku_id,memory_type_id,design_type_id)
			head_title_rs = execute_query(sql,val)

			if head_title_rs != ():
				head_title = str(head_title_rs[0][0])+"-"+str(head_title_rs[0][1])+"-"+str(head_title_rs[0][2])+"-"+str(head_title_rs[0][3])

		if boardid is None:
			boardid = 0

		if boardid == '':
			boardid = 0

		sql="SELECT IFNULL(c.CategoryName,''),IFNULL(b.ComponentName,''),IF(d.UserName='N.A','',d.UserName),a.SecondaryWWID,IFNULL(a.PrimaryWWID,0),a.CategoryID,a.ComponentID,IFNULL(IF(e.UserName='N.A','',e.UserName),''),IFNULL(a.UpdatedOn,'') FROM ComponentDesign a LEFT JOIN ComponentType b ON a.ComponentID = b.ComponentID LEFT JOIN CategoryType c ON a.CategoryID = c.CategoryID LEFT JOIN HomeTable d ON a.PrimaryWWID = d.WWID LEFT JOIN HomeTable e ON e.WWID = a.UpdateBy WHERE a.IsValid = %s AND a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s ORDER BY c.CategoryName,b.ComponentName"
		val = (True,platform_id,sku_id,memory_type_id,design_type_id)
		result_target = execute_query(sql,val)

		submit_process = request.form.get("submit_process")
		if submit_process == "import":
			edit_mode = True
			is_imported = True

			import_platform_id = request.form.get("import_platform_id")
			import_sku_id = request.form.get("import_sku_id")
			import_memory_type_id = request.form.get("import_memory_type_id")
			import_design_type_id = request.form.get("import_design_type_id")

			import_comp_list = []
			for i in range(0,len(result_target)):
				import_comp_list.append(result_target[i][6])

			import_comp_list = set(import_comp_list)
			import_comp_list = list(import_comp_list)

			if import_comp_list != []:
				sql="SELECT IFNULL(c.CategoryName,''),IFNULL(b.ComponentName,''),IF(d.UserName='N.A','',d.UserName),a.SecondaryWWID,IFNULL(a.PrimaryWWID,0),a.CategoryID,a.ComponentID,IFNULL(IF(e.UserName='N.A','',e.UserName),''),IFNULL(a.UpdatedOn,'') FROM ComponentDesign a LEFT JOIN ComponentType b ON a.ComponentID = b.ComponentID LEFT JOIN CategoryType c ON a.CategoryID = c.CategoryID LEFT JOIN HomeTable d ON a.PrimaryWWID = d.WWID LEFT JOIN HomeTable e ON e.WWID = a.UpdateBy WHERE a.IsValid = %s AND a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s AND a.ComponentID IN %s ORDER BY c.CategoryName,b.ComponentName"
				val = (True,import_platform_id,import_sku_id,import_memory_type_id,import_design_type_id,import_comp_list)
				result = execute_query(sql,val)
			else:
				result = ()

		else:
			result = copy.deepcopy(result_target)

		if result !=():

			board_name = result[0][0]

			for i in range(len(result)):

				temp_sec_wwid_list = []
				data_temp_list = []
				data_temp_list.append(result[i][0])	# category name
				data_temp_list.append(result[i][1])	# component name
				data_temp_list.append(result[i][2].replace(',',''))	# primary owner name

				SecondaryOwner = ''
				if(result[i][3] != None):
					rem = result[i][3][1:-1]
					if(rem != None or  rem != "" ):
						spl = rem.split(",")
						if (spl != ['']):
							for j in range(0,len(spl)):
								lead1_wwid = (spl[j])
								if len(lead1_wwid) >= 8:
									
									temp_sec_wwid_list.append(int(lead1_wwid))

									if str(lead1_wwid) != str(99999999):
										sql = "SELECT UserName FROM HomeTable WHERE WWID = %s"
										val=(str(lead1_wwid),)
										name = execute_query(sql,val)

										if name != ():
											if SecondaryOwner == '':
												SecondaryOwner = name[0][0]
											else:
												SecondaryOwner = SecondaryOwner + '<br>' + name[0][0]

				# to remove duplicates wwids
				temp_sec_wwid_list = set(temp_sec_wwid_list)
				temp_sec_wwid_list = list(temp_sec_wwid_list)

				data_temp_list.append(SecondaryOwner)		# sec owner name list for display
				data_temp_list.append(result[i][4])		# primary owner wwid
				data_temp_list.append(temp_sec_wwid_list)	# sec owner wwid list
				data_temp_list.append(result[i][5])		# category id
				data_temp_list.append(result[i][6])		# component id
				data_temp_list.append(result[i][7])		# updated by
				data_temp_list.append(result[i][8])		# updated on

				data.append(data_temp_list)
		else:
			error_message = 'No details are found'

	else:
		pass

	return render("design_owners.html",head_title=head_title,is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,username=username,user_role_name=user_role_name,region_name=region_name,is_imported=is_imported,error_message=error_message,edit_mode=edit_mode,edit_access=edit_access,is_admin=is_admin,user_list=user_list,role_list=role_list,platform_id=platform_id,sku_id=sku_id,memory_type_id=memory_type_id,design_type_id=design_type_id,sku=sku,plat=plat,mem=mem,des=des,design_status_list=design_status_list,design_list=design_list,board_name=board_name,boardid=boardid,data=data)


@app.route("/update_design_owner_details",methods = ['POST', 'GET'])
def update_design_owner_details():

	wwid=session.get('wwid')
	username = session.get('username')

	max_row_count = request.form.get("max_row_count")

	boardid = request.form.get("boardid")
	platform = request.form.get("platform")
	sku = request.form.get("sku")
	memory_type = request.form.get("memory_type")
	design_type = request.form.get("design_type")
	is_imported = request.form.get("is_imported")


	sql="SELECT SKUName FROM SUK WHERE SKUID = %s"
	val =(sku,)
	rs_sku=execute_query(sql,val)

	sku_name = ''
	if rs_sku != ():
		sku_name = rs_sku[0][0]

	sql="SELECT PlatformName FROM Platform WHERE PlatformID = %s"
	val =(platform,)
	rs_plat=execute_query(sql,val)

	plat_name = ''
	if rs_plat != ():
		plat_name = rs_plat[0][0]

	sql="SELECT MemTypeName FROM MemType WHERE MemTypeID = %s"
	val =(memory_type,)
	rs_mem=execute_query(sql,val)

	mem_name = ''
	if rs_mem != ():
		mem_name = rs_mem[0][0]

	sql="SELECT DesignTypeName FROM DesignType WHERE DesignTypeID = %s"
	val =(design_type,)
	rs_des=execute_query(sql,val)

	des_name = ''
	if rs_des != ():
		des_name = rs_des[0][0]

	sql = "SELECT DISTINCT WWID,UserName,RoleID,AdminAccess,EmailID FROM HomeTable WHERE WWID <> 99999999 ORDER BY UserName"
	user_list_rs = execute_query_sql(sql)

	user_list = []
	for row in user_list_rs:
		user_list.append([row[0],row[1],row[2],row[3],row[4]])

	sql = "SELECT IFNULL(c.CategoryName,'') as CatName,IFNULL(b.ComponentName,'') as CompName,IF(d.UserName='N.A','',d.UserName) as PriOwner,a.SecondaryWWID as SecOwnerWWID,IFNULL(a.PrimaryWWID,0) as PriOwnerWWID,a.CategoryID as CatID,a.ComponentID as CompID,IFNULL(IF(e.UserName='N.A','',e.UserName),'') as UpdatedByUser,IFNULL(a.UpdatedOn,'') as UpdateOnDate,IFNULL(f.CategoryLeadWWID,'') as PriCatOwnerWWID,IFNULL(f.CategoryLeadWWID1,'[]') as SecCatOwnerWWID,IFNULL(IF(g.UserName='N.A','',g.UserName),'') as PriCatOwner,FALSE as EditFlag FROM ComponentDesign a LEFT JOIN ComponentType b ON a.ComponentID = b.ComponentID LEFT JOIN CategoryType c ON a.CategoryID = c.CategoryID LEFT JOIN HomeTable d ON a.PrimaryWWID = d.WWID LEFT JOIN HomeTable e ON e.WWID = a.UpdateBy LEFT JOIN CategoryLeadTable f ON a.PlatformID = f.PlatformID AND a.SKUID = f.SKUID AND a.MemTypeID = f.MemTypeID AND a.DesignTypeID = f.DesignTypeID AND a.CategoryID = f.CategoryID LEFT JOIN HomeTable g ON f.CategoryLeadWWID = g.WWID WHERE a.IsValid = %s AND a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s ORDER BY CatName ASC,CompName ASC"
	val = (True,platform,sku,memory_type,design_type)
	results_before_update=execute_query(sql,val)

	if max_row_count is None:
		max_row_count = 0

	for i in range(0,int(max_row_count)):
		category_id = request.form.get("category_id_"+str(i))
		component_id = request.form.get("component_id_"+str(i))
		primary_owner = request.form.get("primary_owner_"+str(i))
		sec_owner = request.form.getlist("sec_owner_"+str(i))

		comp_checkbox = request.form.get("comp_checkbox_"+str(i))

		is_valid_update = False

		if is_imported == "False":
			is_valid_update = True

		if (is_imported == "True") and comp_checkbox:
			is_valid_update = True

		if is_valid_update:

			sec_owner = [int(x) for x in sec_owner]

			if sec_owner == []:
				sec_owner = [99999999]

			if primary_owner is not None:
				sql="SELECT ComponentName FROM ComponentType WHERE ComponentID = %s"
				val =(component_id,)
				rs_comp=execute_query(sql,val)

				comp_name = ''
				if rs_comp != ():
					comp_name = rs_comp[0][0]

				# log table
				try:
					log_notes = 'User has updated Design Owner details for Interface: '+str(comp_name)+'<br>SKU: '+str(sku_name)+'<br>Platform: '+str(plat_name)+'<br>Memory Type: '+str(mem_name)+'<br>Design Type: '+str(des_name)
					log_notes += '<br><br>Primary Owner: '+str(primary_owner)+'<br>Secondary Owner: '+str(sec_owner)
					log_wwid = session.get('wwid')
					t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
					sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
					val = ('Design Owners',0,0,component_id,log_wwid,t,log_notes)
					execute_query(sql,val)
				except:
					if is_logging:
						logging.exception('')
					print("log error.")

				sql = "SELECT * FROM ComponentDesign WHERE ComponentID = %s AND CategoryID = %s AND PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s"
				val = (component_id,category_id,platform,sku,memory_type,design_type)
				result = execute_query(sql,val)

				if result == ():
					sql = "INSERT INTO ComponentDesign(ComponentID,CategoryID,PlatformID,SKUID,PrimaryWWID,SecondaryWWID,SecondaryWWIDTwo,MemTypeID,DesignTypeID,UpdateBy,UpdatedOn,IsValid) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
					val = (component_id,category_id,platform,sku,primary_owner,str(sec_owner),99999999,memory_type,design_type,wwid,t,True)
					execute_query(sql,val)

				else:
					sql = "UPDATE ComponentDesign SET PrimaryWWID = %s, SecondaryWWID = %s, UpdateBy = %s, UpdatedOn = %s, IsValid = %s WHERE ComponentID = %s AND CategoryID = %s AND PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s"
					val=(primary_owner,str(sec_owner),wwid,t,True,component_id,category_id,platform,sku,memory_type,design_type)
					execute_query(sql,val)

	# to sync freeze Design owners for active designs if any change has happened
	sync_design_design_owners(platform=platform,sku=sku,memory_type=memory_type,design_type=design_type)

	# email part
	sql = "SELECT IFNULL(c.CategoryName,'') as CatName,IFNULL(b.ComponentName,'') as CompName,IF(d.UserName='N.A','',d.UserName) as PriOwner,a.SecondaryWWID as SecOwnerWWID,IFNULL(a.PrimaryWWID,0) as PriOwnerWWID,a.CategoryID as CatID,a.ComponentID as CompID,IFNULL(IF(e.UserName='N.A','',e.UserName),'') as UpdatedByUser,IFNULL(a.UpdatedOn,'') as UpdateOnDate,IFNULL(f.CategoryLeadWWID,'') as PriCatOwnerWWID,IFNULL(f.CategoryLeadWWID1,'[]') as SecCatOwnerWWID,IFNULL(IF(g.UserName='N.A','',g.UserName),'') as PriCatOwner,FALSE as EditFlag FROM ComponentDesign a LEFT JOIN ComponentType b ON a.ComponentID = b.ComponentID LEFT JOIN CategoryType c ON a.CategoryID = c.CategoryID LEFT JOIN HomeTable d ON a.PrimaryWWID = d.WWID LEFT JOIN HomeTable e ON e.WWID = a.UpdateBy LEFT JOIN CategoryLeadTable f ON a.PlatformID = f.PlatformID AND a.SKUID = f.SKUID AND a.MemTypeID = f.MemTypeID AND a.DesignTypeID = f.DesignTypeID AND a.CategoryID = f.CategoryID LEFT JOIN HomeTable g ON f.CategoryLeadWWID = g.WWID WHERE a.IsValid = %s AND a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s ORDER BY CatName ASC,CompName ASC"
	val = (True,platform,sku,memory_type,design_type)
	results_after_update=execute_query(sql,val)

	deleted_interfaces = []
	comp_list_aft_update = []
	comp_list_bfr_update = []

	for row in results_after_update:
		comp_list_aft_update.append(row[6])

	for row in results_before_update:
		comp_list_bfr_update.append(row[6])

		if row[6] not in comp_list_aft_update:
			deleted_interfaces.append(row[1])


	html = '<br><table style="width: 100%;border: 1px solid #ddd;padding: 2px;">'
	html += '<tr><td style="width: 5%;border-bottom: 1px solid #ddd;"><b>S.No</b></td><td style="width: 30%;border-bottom: 1px solid #ddd;"><b>Interface Name</b></td><td style="width: 30%;border-bottom: 1px solid #ddd;"><b>Primary Owner</b></td><td style="width: 35%;border-bottom: 1px solid #ddd;"><b>Secondary Owners</b></td></tr>'

	count = 0
	for i in range(0,len(results_after_update)):

		is_found = False
		for j in range(0,len(results_before_update)):

			if results_after_update[i][6] == results_before_update[j][6]:
				is_found = True
				break

		if i > 0:
			if results_after_update[i][5] != results_after_update[i-1][5]:
				html += '<tr><td colspan="4" style="border-bottom: 1px solid #ddd;background-color: #E5F7FF;text-align: center;"><b>'+str(results_after_update[i][0])+'</b></td></tr>'
				count = 0
		else:
			html += '<tr><td colspan="4" style="border-bottom: 1px solid #ddd;background-color: #E5F7FF;text-align: center;"><b>'+str(results_after_update[i][0])+'</b></td></tr>'
			count = 0

		sec_owners = ''
		sec_owners_list = []
		# each row
		if(results_after_update[i][3] != None):
			rem = str(results_after_update[i][3])[1:-1]
			if(rem != None or  rem != "" ):
				spl = rem.split(", ")
				if type(spl) == list:
					for row in user_list:
						if str(row[0]) in spl:
							sec_owners_list.append(row[1])

		sec_owners = '<br>'.join(sec_owners_list)
		count += 1
		comp_color = "black"
		comp_prim_own_color = "black"
		comp_sec_own_color = "black"

		if is_found:
			if results_after_update[i][2] != results_before_update[j][2]:
				comp_prim_own_color = "red"

			if results_after_update[i][3] != results_before_update[j][3]:
				comp_sec_own_color = "red"
		else:
			comp_color = "green"
			comp_prim_own_color = "green"
			comp_sec_own_color = "green"

		html += '<tr><td style="border-bottom: 1px solid #ddd;"><span style="color:'+comp_color+'">'+str(count)+'</span></td><td style="border-bottom: 1px solid #ddd;"><span style="color:'+comp_color+'">'+str(results_after_update[i][1])+'</span></td><td style="border-bottom: 1px solid #ddd;"><span style="color:'+comp_prim_own_color+'">'+str(results_after_update[i][2])+'</span></td><td style="border-bottom: 1px solid #ddd;"><span style="color:'+comp_sec_own_color+'">'+str(sec_owners)+'</span></td></tr>'

	html += '</table>'

	if deleted_interfaces != []:
		html += '<br><br><b>Deleted Interfaces:</b>'
		html += '<br>'.join(deleted_interfaces)

	html += '<br><br>Thanks,<br>ERAM.'

	subject = "Design Owners - Modified"
	message = 'Hi,<br><br><b>'+username+'</b> has updated Design Owner details for,<br><br><font style="color: grey;">Platform: </font><b>'+plat_name+'</b><br><font style="color: grey;">SKU: </font><b>'+sku_name+'</b><br><font style="color: grey;">Memory Type: </font><b>'+mem_name+'</b><br><font style="color: grey;">Design Type: </font><b>'+des_name+'</b><br><br>'
	message += 'Updated details are below,'
	message += html
	email_list = []

	for row in user_list:

		# all admin users
		if row[3] == "yes":
			email_list.append(row[4])

		# who updated
		if row[0] == wwid:
			email_list.append(row[4])

	email_list = sorted(set(email_list), reverse=True)
	email_list = list(email_list)
	
	for i in email_list:
		send_mail_html(i,subject,message,email_list)

	#return redirect(url_for('design_owners', _external=True))
	return design_owners(boardid=boardid,platform_id=platform,sku_id=sku,memory_type_id=memory_type,design_type_id=design_type)

@app.route("/get_plat_sku_design_mem_details",methods = ['POST', 'GET'])
def get_plat_sku_design_mem_details():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]

	result = {}

	final_data =[]

	sql = "SELECT BoardID,PlatformID,SKUID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	rs_details = execute_query(sql,val)

	if rs_details != ():
		final_data.append(rs_details[0][1])
		final_data.append(rs_details[0][2])
		final_data.append(rs_details[0][3])
		final_data.append(rs_details[0][4])

	result["final_data"] = json.dumps(final_data)

	return jsonify(result)


@app.route("/get_import_plat_sku_design_mem_details",methods = ['POST', 'GET'])
def get_import_plat_sku_design_mem_details():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]

	result = {}

	final_data =[]

	sql = "SELECT BoardID,PlatformID,SKUID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	rs_details = execute_query(sql,val)

	if rs_details != ():
		final_data.append(rs_details[0][1])
		final_data.append(rs_details[0][2])
		final_data.append(rs_details[0][3])
		final_data.append(rs_details[0][4])

	result["final_data"] = json.dumps(final_data)

	return jsonify(result)

@app.route("/get_design_for_owners",methods = ['POST', 'GET'])
def get_design_for_owners():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	platform_id = data["platform_id"]
	sku_id = data["sku_id"]
	memory_type_id = data["memory_type_id"]
	design_type_id = data["design_type_id"]

	result = {}

	final_data =[]

	sql = "SELECT BoardID FROM BoardDetails WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s ORDER BY BoardID DESC LIMIT 1"
	val = (platform_id,sku_id,memory_type_id,design_type_id)
	rs_details = execute_query(sql,val)

	if rs_details != ():
		final_data.append(rs_details[0][0])

	result["final_data"] = json.dumps(final_data)

	return jsonify(result)


@app.route('/electrical_owners', methods=['POST', 'GET'])
def electrical_owners(boardid=0,platform_id=0,sku_id=0,memory_type_id=0,design_type_id=0):

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'electrical_owners'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	wwid =  session.get('wwid')
	username =  session.get('username')
	user_role_name =  session.get('user_role_name')
	region_name = session.get('region_name')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	like_wwid = '%' + str(wwid) + '%'

	board_name = ''

	'''
	boardid = 0
	platform_id = 0
	sku_id = 0
	memory_type_id = 0
	design_type_id = 0
	'''

	data = []
	error_message = ''
	edit_access = False
	edit_access_for_cat_lead = False
	is_imported = False
	#cat_level_edit_access = False
	#cat_level_edit_access_list = []
	head_title = ""

	pif_lead_names = ''
	pif_leads_wwid = []

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	is_admin = False
	if has_admin_access == "yes":
		is_admin = True
		edit_access = True

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	# only admin, design lead & manager can have edit access
	#if(role == 10 or role == 12 or role == 1):
	#	edit_access = True

	if (role in [1,2,4,10,12,14]) or is_admin:
		pass
	else:
		return render('error_custom.html',error='You do not have access to this page. Please contact admin.',username=username,user_role_name=user_role_name,region_name=region_name)

	#sql="SELECT SKUID,SKUName FROM SUK ORDER BY SKUName"
	sql="SELECT SKUID,SKUName FROM SUK WHERE SKUID <> 27 ORDER BY SKUName"
	sku=execute_query_sql(sql)

	sql="SELECT PlatformID,PlatformName FROM Platform ORDER BY PlatformName"
	plat=execute_query_sql(sql)

	sql="SELECT MemTypeID,MemTypeName FROM MemType ORDER BY MemTypeName"
	mem=execute_query_sql(sql)

	sql="SELECT DesignTypeID,DesignTypeName FROM DesignType ORDER BY DesignTypeName"
	des=execute_query_sql(sql)

	sql = "SELECT DISTINCT WWID,UserName,RoleID FROM HomeTable WHERE IsActive = 1 AND RoleID IN (1,2,4,10,12,14) AND UserName <> 'N.A' ORDER BY RoleID,UserName"
	user_list = execute_query_sql(sql)

	sql = "SELECT DISTINCT RoleID,RoleName FROM RoleTable WHERE RoleID IN (1,2,4,10,12,14) ORDER BY FIELD(RoleID,4,10,12,1,2,14)"
	role_list = execute_query_sql(sql)

	#sql = "SELECT DISTINCT RoleID,RoleName FROM RoleTable WHERE RoleID IN (1,2,4,10,12,14) ORDER BY FIELD(RoleID,10,1,12,4,2,14,12)"
	sql = "SELECT DISTINCT RoleID,RoleName FROM RoleTable WHERE RoleID IN (1,10,12) ORDER BY FIELD(RoleID,10,1,12)"
	cat_role_list = execute_query_sql(sql)

	temp_data = []
	design_status_list = []
	design_list = []
	ongoing_designs_comp_list = []
	active_comp_list = []

	edit_mode = False

	temp_data = get_all_designs_owners()
	#design_status_list = get_status_list_sorted(data_list=temp_data[0])
	design_status_list = get_order_status_list(list=temp_data[0])
	design_list = temp_data[1]

	if request.method == 'POST':

		if (platform_id is None) or (str(platform_id) == str(0)):
			boardid = request.form.get("boardid")
			platform_id = request.form.get("platform_id")
			sku_id = request.form.get("sku_id")
			memory_type_id = request.form.get("memory_type_id")
			design_type_id = request.form.get("design_type_id")

			sql = "SELECT * FROM (SELECT a.PlatformName FROM Platform a WHERE a.PlatformID = %s) AS platform, (SELECT a.SKUName FROM SUK a WHERE a.SKUID = %s) AS sku, (SELECT a.MemTypeName FROM MemType a WHERE a.MemTypeID = %s) AS mem_type, (SELECT a.DesignTypeName FROM DesignType a WHERE a.DesignTypeID = %s) AS design_type"
			val = (platform_id,sku_id,memory_type_id,design_type_id)
			head_title_rs = execute_query(sql,val)

			if head_title_rs != ():
				head_title = str(head_title_rs[0][0])+"-"+str(head_title_rs[0][1])+"-"+str(head_title_rs[0][2])+"-"+str(head_title_rs[0][3])

		if boardid is None:
			boardid = 0

		if boardid == '':
			boardid = 0

		submit_process = request.form.get("submit_process")
		add_comp = request.form.getlist("add_comp")

		'''
		# for enabling edit access for category wise
		sql = "SELECT a.CategoryID FROM CategoryLeadTable a WHERE a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s AND (a.CategoryLeadWWID = %s OR a.CategoryLeadWWID1 LIKE %s)"
		val = (platform_id,sku_id,memory_type_id,design_type_id,wwid,like_wwid)
		result_edit_access=execute_query(sql,val)

		if result_edit_access != ():
			cat_level_edit_access = True

			for row in result_edit_access:
				cat_level_edit_access_list.append(str(row[0]))

		'''

		valid_pif_user = False
		sql = "SELECT * FROM PifLeadTable a WHERE a.PlatformID = %s AND (a.PrimaryOwner = %s OR a.SecondaryOwner LIKE %s)"
		val = (platform_id,wwid,like_wwid)
		#print("val: ",val)
		pif_details_rs = execute_query(sql,val)

		if pif_details_rs != ():
			valid_pif_user = True
			edit_access = True

		# for enabling delete option for edit mode
		sql = "SELECT c.ComponentID FROM BoardDetails a LEFT JOIN ScheduleTable b ON a.BoardID = b.BoardID LEFT JOIN BoardReviewDesigner c ON a.BoardID=c.BoardID WHERE a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s AND b.ScheduleStatusID = %s"
		val = (platform_id,sku_id,memory_type_id,design_type_id,2)
		result=execute_query(sql,val)

		for row in result:
			ongoing_designs_comp_list.append(row[0])

		# for disabling select option in owners list dropdown for edit mode
		sql = "SELECT c.ComponentID FROM BoardDetails a LEFT JOIN ScheduleTable b ON a.BoardID = b.BoardID LEFT JOIN BoardReviewDesigner c ON a.BoardID=c.BoardID WHERE a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s AND (b.ScheduleStatusID = %s OR b.ScheduleStatusID = %s)"
		val = (platform_id,sku_id,memory_type_id,design_type_id,2,3)
		result=execute_query(sql,val)

		for row in result:
			active_comp_list.append(row[0])

		sql = "SELECT * FROM (SELECT IFNULL(c.CategoryName,'') as CatName,IFNULL(b.ComponentName,'') as CompName,IF(d.UserName='N.A','',d.UserName) as PriOwner,a.SecondaryWWID as SecOwnerWWID,IFNULL(a.PrimaryWWID,0) as PriOwnerWWID,a.CategoryID as CatID,a.ComponentID as CompID,IFNULL(IF(e.UserName='N.A','',e.UserName),'') as UpdatedByUser,IFNULL(a.UpdatedOn,'') as UpdateOnDate,IFNULL(f.CategoryLeadWWID,'') as PriCatOwnerWWID,IFNULL(f.CategoryLeadWWID1,'[]') as SecCatOwnerWWID,IFNULL(IF(g.UserName='N.A','',g.UserName),'') as PriCatOwner,FALSE as EditFlag FROM ComponentReview a LEFT JOIN ComponentType b ON a.ComponentID = b.ComponentID LEFT JOIN CategoryType c ON a.CategoryID = c.CategoryID LEFT JOIN HomeTable d ON a.PrimaryWWID = d.WWID LEFT JOIN HomeTable e ON e.WWID = a.UpdateBy LEFT JOIN CategoryLeadTable f ON a.PlatformID = f.PlatformID AND a.SKUID = f.SKUID AND a.MemTypeID = f.MemTypeID AND a.DesignTypeID = f.DesignTypeID AND a.CategoryID = f.CategoryID LEFT JOIN HomeTable g ON f.CategoryLeadWWID = g.WWID WHERE a.IsValid = %s AND a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s) as Y"
		val = (True,platform_id,sku_id,memory_type_id,design_type_id)

		if add_comp != []:
			sql += " UNION (SELECT IFNULL(b.CategoryName,'') as CatName,IFNULL(a.ComponentName,'') as CompName,'' as PriOwner,'[99999999]' as SecOwnerWWID,0 as PriOwnerWWID,a.CategoryID as CatID,a.ComponentID as CompID,'' as UpdatedByUser,'' as UpdateOnDate,'' as PriCatOwnerWWID,'[]' as SecCatOwnerWWID,'' as PriCatOwner,TRUE as EditFlag FROM ComponentType a LEFT JOIN CategoryType b ON a.CategoryID = b.CategoryID WHERE a.ComponentID IN %s)"
			val += (tuple(add_comp),)
			edit_mode = True

		sql += " ORDER BY CatName ASC,EditFlag ASC,CompName ASC"
		if submit_process == "import":
			edit_mode = True
			is_imported = True

			import_platform_id = request.form.get("import_platform_id")
			import_sku_id = request.form.get("import_sku_id")
			import_memory_type_id = request.form.get("import_memory_type_id")
			import_design_type_id = request.form.get("import_design_type_id")

			val = (True,import_platform_id,import_sku_id,import_memory_type_id,import_design_type_id)
			

		result=execute_query(sql,val)

		if result !=():

			board_name = result[0][0]

			for i in range(len(result)):

				temp_sec_wwid_list = []
				temp_cat_wwid_list = []
				data_temp_list = []
				data_temp_list.append(result[i][0])	# category name
				data_temp_list.append(result[i][1])	# component name
				data_temp_list.append(result[i][2].replace(',',''))	# primary owner name

				SecondaryOwner = ''
				if(result[i][3] != None):
					rem = result[i][3][1:-1]
					if(rem != None or  rem != "" ):
						spl = rem.split(",")
						if (spl != ['']):
							for j in range(0,len(spl)):
								lead1_wwid = (spl[j])
								if len(lead1_wwid) >= 8:
									
									temp_sec_wwid_list.append(int(lead1_wwid))

									if str(lead1_wwid) != str(99999999):
										sql = "SELECT UserName FROM HomeTable WHERE WWID = %s"
										val=(str(lead1_wwid),)
										name = execute_query(sql,val)

										if name != ():
											if SecondaryOwner == '':
												SecondaryOwner = name[0][0]
											else:
												SecondaryOwner = SecondaryOwner + '<br>' + name[0][0]


				CategoryOwner = '-'

				# for appending primary category owner to secondary owner
				if result[i][11] != '':
					CategoryOwner = copy.deepcopy(result[i][11].replace(',',' '))

				#for i in range(0,len(result[i][6])):
				if(result[i][10] != None):
					rem = result[i][10][1:-1]
					if(rem != None or  rem != "" ):
						spl = rem.split(",")
						if (spl != ['']):
							for j in range(0,len(spl)):
								lead1_wwid = (spl[j])
								if len(lead1_wwid) >= 8:
									
									temp_cat_wwid_list.append(int(lead1_wwid))

									if str(lead1_wwid) != str(99999999):
										sql = "SELECT UserName FROM HomeTable WHERE WWID = %s"
										val=(str(lead1_wwid),)
										name = execute_query(sql,val)

										if name != ():
											if CategoryOwner == '-':
												CategoryOwner = name[0][0]
											else:
												CategoryOwner = CategoryOwner + ';&nbsp;&nbsp;&nbsp;' + name[0][0]

				# to remove duplicates wwids
				temp_sec_wwid_list = set(temp_sec_wwid_list)
				temp_sec_wwid_list = list(temp_sec_wwid_list)

				if result[i][9] != '':
					temp_cat_wwid_list.append(int(result[i][9]))

				# to remove duplicates wwids
				temp_cat_wwid_list = set(temp_cat_wwid_list)
				temp_cat_wwid_list = list(temp_cat_wwid_list)

				data_temp_list.append(SecondaryOwner)		# sec owner name list for display
				data_temp_list.append(result[i][4])		# primary owner wwid
				data_temp_list.append(temp_sec_wwid_list)	# sec owner wwid list
				data_temp_list.append(result[i][5])		# category id
				data_temp_list.append(result[i][6])		# component id
				data_temp_list.append(result[i][7])		# updated by
				data_temp_list.append(result[i][8])		# updated on
				data_temp_list.append(CategoryOwner)		# Category owner name list for display
				data_temp_list.append(temp_cat_wwid_list)	# Category owner wwid list

				# edit mode for row based - 12
				if submit_process == "import":
					# access control for only admin / pif leads / respective category leads
					if is_admin or valid_pif_user or (int(wwid) in temp_cat_wwid_list):
						data_temp_list.append(True)
					else:
						data_temp_list.append(False)
				else:
					data_temp_list.append(result[i][12])

				# 13 - for enabling delete option - if any ongoing design having this Interface then block delete option
				if result[i][6] in ongoing_designs_comp_list:
					data_temp_list.append(False)
				else:
					if is_imported:
						data_temp_list.append(False)
					else:
						data_temp_list.append(True)

				# 14 - for disabling select option - if any ongoing and yet to kickstart design having this Interface then block select option, so that owner should be selected
				if result[i][6] in active_comp_list:
					data_temp_list.append(False)
				else:
					data_temp_list.append(True)

				# 15 - edit mode for category owners
				if int(wwid) in temp_cat_wwid_list:
					data_temp_list.append(True)
					edit_access_for_cat_lead = True
				else:
					data_temp_list.append(False)

				data.append(data_temp_list)
		else:
			error_message = 'No details are found'

		# pif lead names
		sql = "SELECT a.PlatformID,a.PlatformName,IFNULL(c.UserName,''),IFNULL(b.SecondaryOwner,''),IFNULL(b.PrimaryOwner,0) FROM Platform a LEFT JOIN PifLeadTable b ON a.PlatformID = b.PlatformID LEFT JOIN HomeTable c ON b.PrimaryOwner = c.WWID WHERE a.PlatformID = %s"
		val = (platform_id,)
		result=execute_query(sql,val)

		if result != ():
			pif_lead_names = result[0][2].replace(",","")

		for i in range(len(result)):

			if(result[i][3] != None):
				rem = result[i][3][1:-1]
				if(rem != None or  rem != "" ):
					spl = rem.split(",")
					if (spl != ['']):
						for j in range(0,len(spl)):
							lead1_wwid = (spl[j])
							if len(lead1_wwid) >= 8:

								if str(lead1_wwid) != str(99999999):
									sql = "SELECT UserName FROM HomeTable WHERE WWID = %s"
									val=(str(lead1_wwid),)
									name = execute_query(sql,val)

									pif_leads_wwid.append(str(lead1_wwid))

									if name != ():
										if pif_lead_names == '':
											pif_lead_names = name[0][0].replace(",","")
										else:
											pif_lead_names = pif_lead_names + ';&nbsp;&nbsp;&nbsp;' + name[0][0].replace(",","")

	else:
		pass

	if pif_lead_names == '':
		pif_lead_names = ' - '

	if str(wwid) in pif_leads_wwid:
		edit_access = True

	return render("electrical_owners.html",head_title=head_title,edit_access_for_cat_lead=edit_access_for_cat_lead,pif_lead_names=pif_lead_names,is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,user_role_name=user_role_name,cat_role_list=cat_role_list,is_imported=is_imported,error_message=error_message,edit_mode=edit_mode,edit_access=edit_access,is_admin=is_admin,user_list=user_list,role_list=role_list,platform_id=platform_id,sku_id=sku_id,memory_type_id=memory_type_id,design_type_id=design_type_id,sku=sku,plat=plat,mem=mem,des=des,username=username,design_status_list=design_status_list,design_list=design_list,region_name=region_name,board_name=board_name,boardid=int(boardid),data=data)


@app.route("/update_elec_owner_details",methods = ['POST', 'GET'])
def update_elec_owner_details():

	wwid=session.get('wwid')
	username = session.get('username')

	max_row_count = request.form.get("max_row_count")

	boardid = request.form.get("boardid")
	platform = request.form.get("platform")
	sku = request.form.get("sku")
	memory_type = request.form.get("memory_type")
	design_type = request.form.get("design_type")

	is_imported = request.form.get("is_imported")

	sql="SELECT SKUName FROM SUK WHERE SKUID = %s"
	val =(sku,)
	rs_sku=execute_query(sql,val)

	sku_name = ''
	if rs_sku != ():
		sku_name = rs_sku[0][0]

	sql="SELECT PlatformName FROM Platform WHERE PlatformID = %s"
	val =(platform,)
	rs_plat=execute_query(sql,val)

	plat_name = ''
	if rs_plat != ():
		plat_name = rs_plat[0][0]

	sql="SELECT MemTypeName FROM MemType WHERE MemTypeID = %s"
	val =(memory_type,)
	rs_mem=execute_query(sql,val)

	mem_name = ''
	if rs_mem != ():
		mem_name = rs_mem[0][0]

	sql="SELECT DesignTypeName FROM DesignType WHERE DesignTypeID = %s"
	val =(design_type,)
	rs_des=execute_query(sql,val)

	des_name = ''
	if rs_des != ():
		des_name = rs_des[0][0]

	# to get complete user list
	sql = "SELECT DISTINCT WWID,UserName,RoleID,AdminAccess,EmailID FROM HomeTable WHERE WWID <> 99999999 ORDER BY UserName"
	user_list_rs = execute_query_sql(sql)

	user_list = []
	for row in user_list_rs:
		user_list.append([row[0],row[1],row[2],row[3],row[4]])

	sql = "SELECT IFNULL(c.CategoryName,'') as CatName,IFNULL(b.ComponentName,'') as CompName,IF(d.UserName='N.A','',d.UserName) as PriOwner,a.SecondaryWWID as SecOwnerWWID,IFNULL(a.PrimaryWWID,0) as PriOwnerWWID,a.CategoryID as CatID,a.ComponentID as CompID,IFNULL(IF(e.UserName='N.A','',e.UserName),'') as UpdatedByUser,IFNULL(a.UpdatedOn,'') as UpdateOnDate,IFNULL(f.CategoryLeadWWID,'') as PriCatOwnerWWID,IFNULL(f.CategoryLeadWWID1,'[]') as SecCatOwnerWWID,IFNULL(IF(g.UserName='N.A','',g.UserName),'') as PriCatOwner,FALSE as EditFlag FROM ComponentReview a LEFT JOIN ComponentType b ON a.ComponentID = b.ComponentID LEFT JOIN CategoryType c ON a.CategoryID = c.CategoryID LEFT JOIN HomeTable d ON a.PrimaryWWID = d.WWID LEFT JOIN HomeTable e ON e.WWID = a.UpdateBy LEFT JOIN CategoryLeadTable f ON a.PlatformID = f.PlatformID AND a.SKUID = f.SKUID AND a.MemTypeID = f.MemTypeID AND a.DesignTypeID = f.DesignTypeID AND a.CategoryID = f.CategoryID LEFT JOIN HomeTable g ON f.CategoryLeadWWID = g.WWID WHERE a.IsValid = %s AND a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s ORDER BY CatName ASC,CompName ASC"
	val = (True,platform,sku,memory_type,design_type)
	results_before_update=execute_query(sql,val)

	if max_row_count is None:
		max_row_count = 0

	for i in range(0,int(max_row_count)):
		category_id = request.form.get("category_id_"+str(i))
		component_id = request.form.get("component_id_"+str(i))
		primary_owner = request.form.get("primary_owner_"+str(i))
		sec_owner = request.form.getlist("sec_owner_"+str(i))

		cat_owner_valid = request.form.get("cat_owner_valid_"+str(i))
		cat_owner = request.form.getlist("cat_owner_"+str(i))

		comp_checkbox = request.form.get("comp_checkbox_"+str(i))

		is_valid_update = False

		if is_imported == "False":
			is_valid_update = True

		if (is_imported == "True") and comp_checkbox:
			is_valid_update = True

		if category_id is None:
			is_valid_update = False

		if is_valid_update:
			sec_owner = [int(x) for x in sec_owner]

			# to remove duplicates
			sec_owner = list(set(sec_owner))

			if sec_owner == []:
				sec_owner = [99999999]

			if primary_owner is not None:

				sql="SELECT ComponentName FROM ComponentType WHERE ComponentID = %s"
				val =(component_id,)
				rs_comp=execute_query(sql,val)

				comp_name = ''
				if rs_comp != ():
					comp_name = rs_comp[0][0]

				# log table
				try:
					log_notes = 'User has updated Electrical Owner details for Interface: '+str(comp_name)+'<br>SKU: '+str(sku_name)+'<br>Platform: '+str(plat_name)+'<br>Memory Type: '+str(mem_name)+'<br>Design Type: '+str(des_name)
					log_notes += '<br><br>Primary Owner: '+str(primary_owner)+'<br>Secondary Owner: '+str(sec_owner)
					log_wwid = session.get('wwid')
					t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
					sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
					val = ('Electrical Owners',0,0,component_id,log_wwid,t,log_notes)
					execute_query(sql,val)
				except:
					if is_logging:
						logging.exception('')
					print("log error.")

				sql = "SELECT * FROM ComponentReview WHERE ComponentID = %s AND CategoryID = %s AND PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s"
				val = (component_id,category_id,platform,sku,memory_type,design_type)
				result = execute_query(sql,val)

				if result == ():
					sql = "INSERT INTO ComponentReview(ComponentID,CategoryID,PlatformID,SKUID,PrimaryWWID,SecondaryWWID,SecondaryWWIDTwo,MemTypeID,DesignTypeID,UpdateBy,UpdatedOn,IsValid) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
					val = (component_id,category_id,platform,sku,primary_owner,str(sec_owner),99999999,memory_type,design_type,wwid,t,True)
					execute_query(sql,val)

				else:
					sql = "UPDATE ComponentReview SET PrimaryWWID = %s, SecondaryWWID = %s, UpdateBy = %s, UpdatedOn = %s, IsValid = %s WHERE ComponentID = %s AND CategoryID = %s AND PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s"
					val=(primary_owner,str(sec_owner),wwid,t,True,component_id,category_id,platform,sku,memory_type,design_type)
					execute_query(sql,val)


				sql = "SELECT * FROM ComponentDesign WHERE ComponentID = %s AND CategoryID = %s AND PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s"
				val = (component_id,category_id,platform,sku,memory_type,design_type)
				result = execute_query(sql,val)

				if result == ():
					sql = "INSERT INTO ComponentDesign(ComponentID,CategoryID,PlatformID,SKUID,PrimaryWWID,SecondaryWWID,SecondaryWWIDTwo,MemTypeID,DesignTypeID,IsValid) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
					val = (component_id,category_id,platform,sku,99999999,str([99999999]),99999999,memory_type,design_type,True)
					execute_query(sql,val)

			# to update category owners
			if cat_owner_valid is not None:

				cat_owner = [int(x) for x in cat_owner]

				# to remove duplicates
				print("cat_owner before: ",cat_owner)
				cat_owner = list(set(cat_owner))
				print("cat_owner after:", cat_owner)
				
				if cat_owner == []:
					cat_owner = [99999999]

				sql = "INSERT INTO CategoryLeadTable(PlatformID,SKUID,CategoryID,CategoryLeadWWID,CategoryLeadWWID1,MemTypeID,DesignTypeID) VALUES (%s,%s,%s,%s,%s,%s,%s) ON DUPLICATE KEY UPDATE CategoryLeadWWID = %s, CategoryLeadWWID1 = %s"
				val = (platform,sku,category_id,99999999,str(cat_owner),memory_type,design_type,99999999,str(cat_owner))

				execute_query(sql,val)

	# to sync freeze Electrical owners for active designs if any change has happened
	sync_design_elec_owners(platform=platform,sku=sku,memory_type=memory_type,design_type=design_type)

	# email part
	sql = "SELECT IFNULL(c.CategoryName,'') as CatName,IFNULL(b.ComponentName,'') as CompName,IF(d.UserName='N.A','',d.UserName) as PriOwner,a.SecondaryWWID as SecOwnerWWID,IFNULL(a.PrimaryWWID,0) as PriOwnerWWID,a.CategoryID as CatID,a.ComponentID as CompID,IFNULL(IF(e.UserName='N.A','',e.UserName),'') as UpdatedByUser,IFNULL(a.UpdatedOn,'') as UpdateOnDate,IFNULL(f.CategoryLeadWWID,'') as PriCatOwnerWWID,IFNULL(f.CategoryLeadWWID1,'[]') as SecCatOwnerWWID,IFNULL(IF(g.UserName='N.A','',g.UserName),'') as PriCatOwner,FALSE as EditFlag FROM ComponentReview a LEFT JOIN ComponentType b ON a.ComponentID = b.ComponentID LEFT JOIN CategoryType c ON a.CategoryID = c.CategoryID LEFT JOIN HomeTable d ON a.PrimaryWWID = d.WWID LEFT JOIN HomeTable e ON e.WWID = a.UpdateBy LEFT JOIN CategoryLeadTable f ON a.PlatformID = f.PlatformID AND a.SKUID = f.SKUID AND a.MemTypeID = f.MemTypeID AND a.DesignTypeID = f.DesignTypeID AND a.CategoryID = f.CategoryID LEFT JOIN HomeTable g ON f.CategoryLeadWWID = g.WWID WHERE a.IsValid = %s AND a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s ORDER BY CatName ASC,CompName ASC"
	val = (True,platform,sku,memory_type,design_type)
	results_after_update=execute_query(sql,val)

	deleted_interfaces = []
	comp_list_aft_update = []
	comp_list_bfr_update = []

	for row in results_after_update:
		comp_list_aft_update.append(row[6])

	for row in results_before_update:
		comp_list_bfr_update.append(row[6])

		if row[6] not in comp_list_aft_update:
			deleted_interfaces.append(row[1])


	html = '<br><table style="width: 100%;border: 1px solid #ddd;padding: 2px;">'
	html += '<tr><td style="width: 5%;border-bottom: 1px solid #ddd;"><b>S.No</b></td><td style="width: 30%;border-bottom: 1px solid #ddd;"><b>Interface Name</b></td><td style="width: 30%;border-bottom: 1px solid #ddd;"><b>Primary Owner</b></td><td style="width: 35%;border-bottom: 1px solid #ddd;"><b>Secondary Owners</b></td></tr>'

	count = 0
	for i in range(0,len(results_after_update)):

		is_found = False
		for j in range(0,len(results_before_update)):

			if results_after_update[i][6] == results_before_update[j][6]:
				is_found = True
				break

		cat_owners = ''
		cat_owners_list = []
		# each row
		if(results_after_update[i][10] != None):
			rem = str(results_after_update[i][10])[1:-1]
			if(rem != None or  rem != "" ):
				spl = rem.split(", ")
				if type(spl) == list:
					for row in user_list:
						if str(row[0]) in spl:
							cat_owners_list.append(row[1])

		cat_owners = '; '.join(cat_owners_list)
		cat_font_color = "black"

		if is_found:
			if results_after_update[i][10] != results_before_update[j][10]:
				cat_font_color = "red"
		else:
			cat_font_color = "red"

		if i > 0:
			if results_after_update[i][5] != results_after_update[i-1][5]:
				html += '<tr><td colspan="4" style="border-bottom: 1px solid #ddd;background-color: #E5F7FF;text-align: center;"><b>'+str(results_after_update[i][0])+'</b><br><span style="color:'+cat_font_color+'">'+str(cat_owners)+'</span></td></tr>'
				count = 0
		else:
			html += '<tr><td colspan="4" style="border-bottom: 1px solid #ddd;background-color: #E5F7FF;text-align: center;"><b>'+str(results_after_update[i][0])+'</b><br><span style="color:'+cat_font_color+'">'+str(cat_owners)+'</span></td></tr>'
			count = 0

		sec_owners = ''
		sec_owners_list = []
		# each row
		if(results_after_update[i][3] != None):
			rem = str(results_after_update[i][3])[1:-1]
			if(rem != None or  rem != "" ):
				spl = rem.split(", ")
				if type(spl) == list:
					for row in user_list:
						if str(row[0]) in spl:
							sec_owners_list.append(row[1])

		sec_owners = '<br>'.join(sec_owners_list)
		count += 1
		comp_color = "black"
		comp_prim_own_color = "black"
		comp_sec_own_color = "black"

		if is_found:
			if results_after_update[i][2] != results_before_update[j][2]:
				comp_prim_own_color = "red"

			if results_after_update[i][3] != results_before_update[j][3]:
				comp_sec_own_color = "red"
		else:
			comp_color = "green"
			comp_prim_own_color = "green"
			comp_sec_own_color = "green"

		html += '<tr><td style="border-bottom: 1px solid #ddd;"><span style="color:'+comp_color+'">'+str(count)+'</span></td><td style="border-bottom: 1px solid #ddd;"><span style="color:'+comp_color+'">'+str(results_after_update[i][1])+'</span></td><td style="border-bottom: 1px solid #ddd;"><span style="color:'+comp_prim_own_color+'">'+str(results_after_update[i][2])+'</span></td><td style="border-bottom: 1px solid #ddd;"><span style="color:'+comp_sec_own_color+'">'+str(sec_owners)+'</span></td></tr>'

	html += '</table>'

	if deleted_interfaces != []:
		html += '<br><br><b>Deleted Interfaces:</b>'
		html += '<br>'.join(deleted_interfaces)

	html += '<br><br>Thanks,<br>ERAM.'

	subject = "Electrical Owners - Modified"
	message = 'Hi,<br><br><b>'+username+'</b> has updated Electrical Owner details for,<br><br><font style="color: grey;">Platform: </font><b>'+plat_name+'</b><br><font style="color: grey;">SKU: </font><b>'+sku_name+'</b><br><font style="color: grey;">Memory Type: </font><b>'+mem_name+'</b><br><font style="color: grey;">Design Type: </font><b>'+des_name+'</b><br><br>'
	message += 'Updated details are below,'
	message += html
	email_list = []

	for row in user_list:

		# all admin users
		if row[3] == "yes":
			email_list.append(row[4])

		# who updated
		if row[0] == wwid:
			email_list.append(row[4])

	email_list = sorted(set(email_list), reverse=True)
	email_list = list(email_list)

	for i in email_list:
		send_mail_html(i,subject,message,email_list)

	#return redirect(url_for('electrical_owners', _external=True))
	return electrical_owners(boardid=boardid,platform_id=platform,sku_id=sku,memory_type_id=memory_type,design_type_id=design_type)


@app.route("/get_elec_owner_comp_list",methods = ['POST', 'GET'])
def get_elec_owner_comp_list():

	wwid=session.get('wwid')
	username = session.get('username')
	is_admin = session.get('is_admin')

	data = json.loads(request.form.get("data"))
	
	platform_id = data["platform"]
	sku_id = data["sku"]
	memory_type_id = data["memory_type"]
	design_type_id = data["design_type"]

	final_data =[]
	temp_cat_list = []
	temp_data = []
	cat_list = []
	valid_category_id_list = []
	valid_pif_user = False

	like_wwid = '%' + str(wwid) + '%'

	sql = "SELECT * FROM PifLeadTable a WHERE a.PlatformID = %s AND (a.PrimaryOwner = %s OR a.SecondaryOwner LIKE %s)"
	val = (platform_id,wwid,like_wwid)
	pif_details_rs = execute_query(sql,val)

	if pif_details_rs != ():
		valid_pif_user = True

	sql = "SELECT a.CategoryID FROM CategoryLeadTable a WHERE a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s AND (a.CategoryLeadWWID = %s OR a.CategoryLeadWWID1 LIKE %s)"
	val = (platform_id,sku_id,memory_type_id,design_type_id,wwid,like_wwid)
	cat_details_rs = execute_query(sql,val)

	for i in range(len(cat_details_rs)):
		valid_category_id_list.append(int(cat_details_rs[i][0]))

	sql = "SELECT a.ComponentID,a.ComponentName,b.CategoryID,b.CategoryName FROM ComponentType a LEFT JOIN CategoryType b ON a.CategoryID = b.CategoryID WHERE a.ComponentID NOT IN (SELECT x.ComponentID FROM ComponentReview x WHERE x.IsValid = %s AND x.PlatformID = %s AND x.SKUID = %s AND x.MemTypeID = %s AND x.DesignTypeID = %s ORDER BY x.ComponentID) ORDER BY b.CategoryName,a.ComponentName"
	val = (True,platform_id,sku_id,memory_type_id,design_type_id)
	rs_details = execute_query(sql,val)

	for i in range(0,len(rs_details)):

		temp_data.append([rs_details[i][0],rs_details[i][1],rs_details[i][2],rs_details[i][3]])

		if rs_details[i][2] not in temp_cat_list:
			# access control for only admin / pif leads / respective category leads
			if is_admin or valid_pif_user or (int(rs_details[i][2]) in valid_category_id_list):
				temp_cat_list.append(rs_details[i][2])
				cat_list.append([rs_details[i][2],rs_details[i][3]])
	
	final_data.append(cat_list)
	final_data.append(temp_data)

	return jsonify(final_data)

@app.route("/delete_interface_for_elect_owner",methods = ['POST', 'GET'])
def delete_interface_for_elect_owner():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))

	comp_id = copy.deepcopy(data['comp_id'])
	platform = copy.deepcopy(data['platform'])
	sku = copy.deepcopy(data['sku'])
	mem_type = copy.deepcopy(data['mem_type'])
	design_type = copy.deepcopy(data['design_type'])

	# log table
	try:
		log_notes = 'User has Deleted Interface for Electrical Owner <br>Component ID: '+str(comp_id)+'PlatformID: '+str(platform)+'<br>SKU ID: '+str(sku)+'<br>Memory Type ID: '+str(mem_type)+'<br>Desig Type ID: '+str(design_type)+'<br>'
		log_wwid = session.get('wwid')
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
		val = ('Electrical Owners',0,0,comp_id,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")	

	sql = "UPDATE ComponentReview SET IsValid = %s WHERE ComponentID = %s AND PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s"
	val  =(False,comp_id,platform,sku,mem_type,design_type)
	result = execute_query(sql,val)

	sql = "UPDATE ComponentDesign SET IsValid = %s WHERE ComponentID = %s AND PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s"
	val  =(False,comp_id,platform,sku,mem_type,design_type)
	result = execute_query(sql,val)

	return jsonify(result)

def sync_design_elec_owners(platform,sku,memory_type,design_type):

	# to freeze electrical and design owners for the Interface during Signoff Designs, incase in future t0 avoid impact if we change owners for the interface
	sql="SELECT a.BoardID,a.PlatformID,a.SKUID,a.MemTypeID,a.DesignTypeID FROM BoardDetails a,ScheduleTable b WHERE a.BoardID = b.BoardID AND b.ScheduleStatusID IN (2,3,6) AND a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s ORDER BY a.BoardID"
	val = (platform,sku,memory_type,design_type)
	bdetails = execute_query(sql,val)

	print(" elect bdetails: ",bdetails)

	for row in bdetails:

		sql="SELECT ComponentID,CategoryID,PlatformID,SKUID,MemTypeID,DesignTypeID,PrimaryWWID,SecondaryWWID FROM ComponentReview WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s ORDER BY ComponentID ASC"
		val = (row[1],row[2],row[3],row[4])
		compdetails = execute_query(sql,val)

		if compdetails != ():

			for comp_row in compdetails:

				sql="SELECT CategoryLeadWWID,CategoryLeadWWID1 FROM CategoryLeadTable WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s AND CategoryID = %s"
				val = (row[1],row[2],row[3],row[4],comp_row[1])
				categorydetails = execute_query(sql,val)

				prim_cad_lead = 99999999
				sec_cad_lead = '[]'

				if categorydetails != ():
					prim_cad_lead = categorydetails[0][0]
					sec_cad_lead = categorydetails[0][1]

				sql = "INSERT INTO DesignElectricalOwners VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s) ON DUPLICATE KEY UPDATE PrimaryCategoryLead = %s, SecondaryCategoryLead = %s, PrimaryElectricalOwner = %s, SecondaryElectricalOwner = %s"
				val = (row[0],comp_row[0],comp_row[1],comp_row[2],comp_row[3],comp_row[4],comp_row[5],prim_cad_lead,sec_cad_lead,comp_row[6],comp_row[7],prim_cad_lead,sec_cad_lead,comp_row[6],comp_row[7])
				execute_query(sql, val)

	return True


def sync_design_design_owners(platform,sku,memory_type,design_type):

	# to freeze electrical and design owners for the Interface during Signoff Designs, incase in future t0 avoid impact if we change owners for the interface
	sql="SELECT a.BoardID,a.PlatformID,a.SKUID,a.MemTypeID,a.DesignTypeID FROM BoardDetails a,ScheduleTable b WHERE a.BoardID = b.BoardID AND b.ScheduleStatusID IN (2,3,6) AND a.PlatformID = %s AND a.SKUID = %s AND a.MemTypeID = %s AND a.DesignTypeID = %s ORDER BY a.BoardID"
	val = (platform,sku,memory_type,design_type)
	bdetails = execute_query(sql,val)

	print(" design bdetails: ",bdetails)

	for row in bdetails:

		sql="SELECT ComponentID,CategoryID,PlatformID,SKUID,MemTypeID,DesignTypeID,PrimaryWWID,SecondaryWWID FROM ComponentDesign WHERE PlatformID = %s AND SKUID = %s AND MemTypeID = %s AND DesignTypeID = %s ORDER BY ComponentID ASC"
		val = (row[1],row[2],row[3],row[4])
		compdetails2 = execute_query(sql,val)

		if compdetails2 != ():

			for comp_row in compdetails2:

				sql = "INSERT INTO DesignDesignOwners VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s) ON DUPLICATE KEY UPDATE PrimaryOwner = %s, SecondaryOwner = %s"
				val = (row[0],comp_row[0],comp_row[1],comp_row[2],comp_row[3],comp_row[4],comp_row[5],comp_row[6],comp_row[7],comp_row[6],comp_row[7])
				execute_query(sql, val)

	return True

@app.route("/test",methods = ['POST', 'GET'])
def test():

	data = {"a": "check","b": "test"}
	print(data)
	return jsonify(data)

@app.route("/test1",methods = ['POST', 'GET'])
def test1():
	print("test 1")

	response = requests.get("http://127.0.0.1:5000/test")
	print(response)
	print(response.status_code)
	#print(response.content()) # Return the raw bytes of the data payload
	#print(response.text()) # Return a string representation of the data payload
	print(response.json()) # This method is convenient when the API returns JSON
	print(response.headers["date"])
	return True

@app.route("/login.html")
def login():

	if is_localhost:
		session['username'] = "Vadivel"
		session['wwid']= "11898148"
		session['sso_loggedin_wwid']= "11898148"
		session['email'] = "vadivelx.balakrishnan@intel.com"
		session['is_admin'] = True
		session['user_role_name'] = "Developer"
		session['is_inactive_user'] = False
		session['inactive_user_msg'] = ""

		if is_prod:
			session['region_name'] = ""
		else:
			session['region_name'] = copy.deepcopy(DB_NAME)

		return redirect(url_for('index', _external=True))
	else:
		return 'Not local host server'

@app.route("/session_clear",methods = ['POST', 'GET'])
def session_clear():

	session.clear()

	return 'Session data cleared'


@app.route("/design_summary",methods = ['POST', 'GET'])
def design_summary():

	if session.get('wwid') is None:
		session['target_page'] = 'design_summary'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	boardid = 0
	if request.method == 'POST':
		boardid = request.form.get("boardid")

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	has_data = False
	active_designs_list = [2,3,6]

	active_designs = []
	completed_designs = []
	platform = []
	sku = []
	mem_type = []
	design_type = []
	temp_board_id = []

	active_designs_temp = []
	completed_designs_temp = []
	platform_temp = []
	sku_temp = []
	mem_type_temp = []
	design_type_temp = []

	sql = "SELECT MIN(ProposedStartDate) FROM DesignCalendar"
	start_date_min = execute_query_sql(sql)[0][0]

	sql = "SELECT MAX(ProposedStartDate) FROM DesignCalendar"
	start_date_max = execute_query_sql(sql)[0][0]

	start_date_default = start_date_min
	end_date_default = start_date_max

	end_date_min = start_date_min
	end_date_max = start_date_max

	start_date_default_ww = get_work_week_fun_with_year(start_date_min)
	end_date_default_ww = get_work_week_fun_with_year(start_date_max)

	sql="SELECT DISTINCT b1.BoardID,b1.BoardName,b1.PlatformID,p1.PlatformName,b1.SKUID,s1.SKUName,b1.MemTypeID,m1.MemTypeName,b1.DesignTypeID,d2.DesignTypeName,s2.ScheduleStatusID FROM BoardDetails b1,Platform p1,SUK s1,MemType m1,DesignType d2,ScheduleTable s2 WHERE s2.BoardID = b1.BoardID AND p1.PlatformID = b1.PlatformID AND b1.SKUID = s1.SKUID AND b1.MemTypeID = m1.MemTypeID AND b1.DesignTypeID = d2.DesignTypeID ORDER BY b1.BoardID DESC"
	result=execute_query_sql(sql)

	if result != ():
		has_data = True

		for i in range(len(result)):
			#designs_temp.append([result[i][0],result[i][1],"checked"])
			if result[i][10] in active_designs_list:
				active_designs_temp.append([result[i][0],result[i][1],"checked"])
			else:
				completed_designs_temp.append([result[i][0],result[i][1],"checked"])

			platform_temp.append([result[i][2],result[i][3],"checked"])
			sku_temp.append([result[i][4],result[i][5],"checked"])
			mem_type_temp.append([result[i][6],result[i][7],"checked"])
			design_type_temp.append([result[i][8],result[i][9],"checked"])

	platform_temp = list(frozenset(tuple(row) for row in platform_temp))
	sku_temp = list(frozenset(tuple(row) for row in sku_temp))
	mem_type_temp = list(frozenset(tuple(row) for row in mem_type_temp))
	design_type_temp = list(frozenset(tuple(row) for row in design_type_temp))

	for row in platform_temp:
		platform.append(list(row))

	for row in sku_temp:
		sku.append(list(row))

	for row in mem_type_temp:
		mem_type.append(list(row))

	for row in design_type_temp:
		design_type.append(list(row))

	for row in active_designs_temp:
		active_designs.append(list(row))									

	for row in completed_designs_temp:
		completed_designs.append(list(row))									

	# sorting
	platform.sort(key = lambda x: x[1])
	sku.sort(key = lambda x: x[1])
	mem_type.sort(key = lambda x: x[1])
	design_type.sort(key = lambda x: x[1])

	#temp = 0
	#for i in designs:
	#	temp += 1

	temp = 0
	for i in platform:
		temp += 1

	platform_all = True if temp > 1 else False

	temp = 0
	for i in sku:
		temp += 1

	sku_all = True if temp > 1 else False

	temp = 0
	for i in mem_type:
		temp += 1

	mem_type_all = True if temp > 1 else False

	temp = 0
	for i in design_type:
		temp += 1

	design_type_all = True if temp > 1 else False

	return render('design_summary.html',start_date_min=start_date_min,start_date_max=start_date_max,start_date_default_ww=start_date_default_ww,end_date_default=end_date_default,end_date_min=end_date_min,end_date_max=end_date_max,end_date_default_ww=end_date_default_ww,start_date_default=start_date_default,boardid=boardid,platform=platform,sku=sku,mem_type=mem_type,design_type=design_type,active_designs=active_designs,completed_designs=completed_designs,platform_all=platform_all,sku_all=sku_all,mem_type_all=mem_type_all,design_type_all=design_type_all,has_data=has_data,username=username,user_role_name=user_role_name,region_name=region_name)

@app.route("/design_summary_data_page",methods = ['POST', 'GET'])
def design_summary_data_page(boardid=0):

	boardid = request.args.get('boardid', default = 0, type = int)

	wwid = session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	sql = "SELECT BoardName, ClosureComment,ReviewTimelineID,RefBoardID,RefBoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	result = execute_query(sql,val)
	boardname = result[0][0]
	closure_comment = result[0][1]
	rid = result[0][2]
	print("rid: ",rid)
	print(type(rid))

	ref_board_name = "-"

	if result[0][4] != "":
		if result[0][3] not in [0,"0"]:
			ref_board_name = "[ID: "+str(result[0][3])+"] - "+str(result[0][4])
		else:
			ref_board_name = str(result[0][4])

	sql = "SELECT ProposedEndDate FROM DesignCalendar WHERE BoardID =%s"
	val = (boardid,)
	enddate = execute_query(sql,val)
	EndDate=[]
	EndDate.append(enddate[0][0])

	if enddate[0][0]:
		ww_end_date = get_work_week_fun_with_year(enddate[0][0])
	else:
		ww_end_date = ''

	EndDate.append(ww_end_date)


	sql = "SELECT a.BoardName,DesignTypeName,BoardStateName,BoardTrackName,ReviewTimelineName,PlatformName,SKUName,MemTypeName,c.Username,DesignManagerWWID,d.Username,e.Username,core,IFNULL(f.RequestID,'') FROM BoardDetails a NATURAL JOIN (DesignType,BoardState,BoardTrack,ReviewTimeline,Platform,SUK,MemType) LEFT JOIN HomeTable c ON a.DesignLeadWWID = c.WWID LEFT JOIN HomeTable d ON a.CADLeadWWID = d.WWID LEFT JOIN HomeTable e ON a.PIFLeadWWID = e.WWID LEFT JOIN RequestMap f ON a.BoardID=f.BoardID WHERE a.BoardID = %s"
	val = (boardid,)
	boarddeets = execute_query(sql,val)
	boarddeets_list = []
	for i in range(len(boarddeets)):
		role = boarddeets[len(boarddeets) - i - 1]
		boarddeets_list.append(role)

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)

	ww_date = []
	if cal[0][3]:
		cal_ww_start_date = get_work_week_fun_with_year(cal[0][3])
	else:
		cal_ww_start_date = ''
	ww_date.append(cal_ww_start_date)

	if cal[0][4]:
		cal_ww_end_date = get_work_week_fun_with_year(cal[0][4])
	else:
		cal_ww_end_date = ''
	ww_date.append(cal_ww_end_date)

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	sql = "SELECT DISTINCT B1.BoardID,B1.BoardID,B1.ComponentID,B1.PDG,B1.CommentDesigner,B1.PDG_Electrical,C1.ComponentName,S2.ScheduleTypeName,H1.UserName,H3.UserName,B1.CommentElectrical,H4.UserName,B1.CommentSignOffInterface,H5.UserName,B1.IsPdgElectricalSubmitted FROM BoardReviewDesigner B1, ComponentType C1, ScheduleTableComponent S1, ScheduleStatusType S2, ComponentReview C2, HomeTable H1, HomeTable H2, HomeTable H3, HomeTable H4, HomeTable H5 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID =%s AND B1.ComponentID = C2.ComponentID AND C2.PrimaryWWID = H1.WWID AND B1.CommentDesignUpdatedBy = H3.WWID AND B1.CommntElectricalUpdatedBy = H4.WWID AND B1.CommentSignOffInterfaceUpdateBy = H5.WWID ORDER BY FIELD(S1.ScheduleStatusID, 3, 6, 2, 1, 4, 5, 7), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
	val = (boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
	comp = execute_query(sql,val)

	status = ""

	sql = "SELECT * FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)
	r = execute_query(sql,val)

	if(len(r)==0):
		status = "Yet to kickoff"
	else:
		sql = "SELECT S1.ScheduleTypeName FROM ScheduleStatusType S1, DesignCalendar D1, ScheduleTable S2 WHERE D1.BoardID = S2.BoardID AND S2.ScheduleStatusID = S1.ScheduleID AND D1.BoardID = %s "
		val = (boardid,)
		status = execute_query(sql,val)[0][0]	

	if rid == 1 and status == "Signed-Off":
		status = "Closed"

	access_admin = 'no'	
	

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	if(has_admin_access == 'yes'):
		access_admin = 'yes'

	sql = "SELECT ht.UserName FROM BoardDetailsRequest br ,RequestMap rm, HomeTable ht WHERE rm.RequestID = br.RequestID AND br.WWID = ht.WWID AND rm.BoardID = %s"
	val = (boardid,)
	design_raised_by = ''

	try:
		design_raised_by = execute_query(sql,val)[0][0]
	except:
		pass

	return render("design_summary_data_page.html",ref_board_name=ref_board_name,rid=rid,username=username,user_role_name=user_role_name,region_name=region_name,BoardID=boardid,design_raised_by=design_raised_by,Boardname=boardname,closure_comment=closure_comment,boarddeets_list=boarddeets_list,cal = cal,comp = comp,Status = status,access_admin = access_admin,EndDate=EndDate,ww_date=ww_date)

@app.route("/get_sku_name_for_summary",methods = ['POST', 'GET'])
def get_sku_name_for_summary():

	array_body = request.form.get("array_body")
	data = json.loads(array_body)
	start_calender_date = data[0]
	end_calender_date = data[1]
	platform_list_input = data[2]

	print("start_calender_date: ",start_calender_date)
	print("end_calender_date: ",end_calender_date)
	final_result = {}

	has_data = False
	active_designs_list = [2,3,6]

	active_designs = []
	completed_designs = []
	platform = []
	sku = []
	mem_type = []
	design_type = []
	temp_board_id = []

	active_designs_temp = []
	completed_designs_temp = []
	platform_temp = []
	sku_temp = []
	mem_type_temp = []
	design_type_temp = []

	sql="SELECT DISTINCT b1.BoardID,b1.BoardName,b1.PlatformID,p1.PlatformName,b1.SKUID,s1.SKUName,b1.MemTypeID,m1.MemTypeName,b1.DesignTypeID,d2.DesignTypeName,s2.ScheduleStatusID FROM BoardDetails b1,Platform p1,SUK s1,MemType m1,DesignType d2,ScheduleTable s2,DesignCalendar d1 WHERE s2.BoardID = b1.BoardID AND p1.PlatformID = b1.PlatformID AND b1.SKUID = s1.SKUID AND b1.MemTypeID = m1.MemTypeID AND b1.DesignTypeID = d2.DesignTypeID AND b1.BoardID = d1.BoardID AND b1.PlatformID IN %s AND d1.ProposedStartDate >= %s AND d1.ProposedStartDate <= %s ORDER BY b1.BoardID DESC"
	val=(platform_list_input,start_calender_date,end_calender_date)
	result=execute_query(sql,val)

	if result != ():
		has_data = True

		for i in range(len(result)):
			#designs_temp.append([result[i][0],result[i][1],"checked"])
			if result[i][10] in active_designs_list:
				active_designs_temp.append([result[i][0],result[i][1],"checked"])
			else:
				completed_designs_temp.append([result[i][0],result[i][1],"checked"])

			platform_temp.append([result[i][2],result[i][3],"checked"])
			sku_temp.append([result[i][4],result[i][5],"checked"])
			mem_type_temp.append([result[i][6],result[i][7],"checked"])
			design_type_temp.append([result[i][8],result[i][9],"checked"])

	platform_temp = list(frozenset(tuple(row) for row in platform_temp)) 
	sku_temp = list(frozenset(tuple(row) for row in sku_temp))
	mem_type_temp = list(frozenset(tuple(row) for row in mem_type_temp))
	design_type_temp = list(frozenset(tuple(row) for row in design_type_temp))

	for row in platform_temp:
		platform.append(list(row))

	for row in sku_temp:
		sku.append(list(row))

	for row in mem_type_temp:
		mem_type.append(list(row))

	for row in design_type_temp:
		design_type.append(list(row))

	for row in active_designs_temp:
		active_designs.append(list(row))									

	for row in completed_designs_temp:
		completed_designs.append(list(row))									

	# sorting
	platform.sort(key = lambda x: x[1])
	sku.sort(key = lambda x: x[1])
	mem_type.sort(key = lambda x: x[1])
	design_type.sort(key = lambda x: x[1])

	temp = 0
	for i in platform:
		temp += 1

	platform_all = True if temp > 1 else False

	temp = 0
	for i in sku:
		temp += 1

	sku_all = True if temp > 1 else False

	temp = 0
	for i in mem_type:
		temp += 1

	mem_type_all = True if temp > 1 else False

	temp = 0
	for i in design_type:
		temp += 1

	design_type_all = True if temp > 1 else False


	final_result['platform'] = platform
	final_result['sku'] = sku
	final_result['mem_type'] = mem_type
	final_result['design_type'] = design_type
	final_result['active_designs'] = active_designs
	final_result['completed_designs'] = completed_designs
	
	final_result['platform_all'] = platform_all
	final_result['sku_all'] = sku_all
	final_result['mem_type_all'] = mem_type_all
	final_result['design_type_all'] = design_type_all

	return jsonify(final_result)

@app.route("/get_mem_type_name_for_summary",methods = ['POST', 'GET'])
def get_mem_type_name_for_summary():

	array_body = request.form.get("array_body")
	data = json.loads(array_body)

	start_calender_date = data[0]
	end_calender_date = data[1]

	print("start_calender_date: ",start_calender_date)
	print("end_calender_date: ",end_calender_date)

	platform_list_input = data[2]
	sku_list_input = data[3]

	final_result = {}

	has_data = False
	active_designs_list = [2,3,6]

	active_designs = []
	completed_designs = []
	platform = []
	sku = []
	mem_type = []
	design_type = []
	temp_board_id = []

	active_designs_temp = []
	completed_designs_temp = []
	platform_temp = []
	sku_temp = []
	mem_type_temp = []
	design_type_temp = []

	sql="SELECT DISTINCT b1.BoardID,b1.BoardName,b1.PlatformID,p1.PlatformName,b1.SKUID,s1.SKUName,b1.MemTypeID,m1.MemTypeName,b1.DesignTypeID,d2.DesignTypeName,s2.ScheduleStatusID FROM BoardDetails b1,Platform p1,SUK s1,MemType m1,DesignType d2,ScheduleTable s2,DesignCalendar d1 WHERE s2.BoardID = b1.BoardID AND p1.PlatformID = b1.PlatformID AND b1.SKUID = s1.SKUID AND b1.MemTypeID = m1.MemTypeID AND b1.DesignTypeID = d2.DesignTypeID AND b1.BoardID = d1.BoardID AND b1.PlatformID IN %s AND b1.SKUID IN %s AND d1.ProposedStartDate >= %s AND d1.ProposedStartDate <= %s ORDER BY b1.BoardID DESC"
	val=(platform_list_input,sku_list_input,start_calender_date,end_calender_date)
	result=execute_query(sql,val)

	if result != ():
		has_data = True

		for i in range(len(result)):
			#designs_temp.append([result[i][0],result[i][1],"checked"])
			if result[i][10] in active_designs_list:
				active_designs_temp.append([result[i][0],result[i][1],"checked"])
			else:
				completed_designs_temp.append([result[i][0],result[i][1],"checked"])

			platform_temp.append([result[i][2],result[i][3],"checked"])
			sku_temp.append([result[i][4],result[i][5],"checked"])
			mem_type_temp.append([result[i][6],result[i][7],"checked"])
			design_type_temp.append([result[i][8],result[i][9],"checked"])

	platform_temp = list(frozenset(tuple(row) for row in platform_temp)) 
	sku_temp = list(frozenset(tuple(row) for row in sku_temp))
	mem_type_temp = list(frozenset(tuple(row) for row in mem_type_temp))
	design_type_temp = list(frozenset(tuple(row) for row in design_type_temp))

	for row in platform_temp:
		platform.append(list(row))

	for row in sku_temp:
		sku.append(list(row))

	for row in mem_type_temp:
		mem_type.append(list(row))

	for row in design_type_temp:
		design_type.append(list(row))

	for row in active_designs_temp:
		active_designs.append(list(row))									

	for row in completed_designs_temp:
		completed_designs.append(list(row))									

	# sorting
	platform.sort(key = lambda x: x[1])
	sku.sort(key = lambda x: x[1])
	mem_type.sort(key = lambda x: x[1])
	design_type.sort(key = lambda x: x[1])

	temp = 0
	for i in platform:
		temp += 1

	platform_all = True if temp > 1 else False

	temp = 0
	for i in sku:
		temp += 1

	sku_all = True if temp > 1 else False

	temp = 0
	for i in mem_type:
		temp += 1

	mem_type_all = True if temp > 1 else False

	temp = 0
	for i in design_type:
		temp += 1

	design_type_all = True if temp > 1 else False


	final_result['platform'] = platform
	final_result['sku'] = sku
	final_result['mem_type'] = mem_type
	final_result['design_type'] = design_type
	final_result['active_designs'] = active_designs
	final_result['completed_designs'] = completed_designs
	
	final_result['platform_all'] = platform_all
	final_result['sku_all'] = sku_all
	final_result['mem_type_all'] = mem_type_all
	final_result['design_type_all'] = design_type_all

	return jsonify(final_result)

@app.route("/get_design_type_name_for_summary",methods = ['POST', 'GET'])
def get_design_type_name_for_summary():

	array_body = request.form.get("array_body")
	data = json.loads(array_body)

	start_calender_date = data[0]
	end_calender_date = data[1]

	print("start_calender_date: ",start_calender_date)
	print("end_calender_date: ",end_calender_date)

	platform_list_input = data[2]
	sku_list_input = data[3]
	mem_type_list_input = data[4]

	final_result = {}

	has_data = False
	active_designs_list = [2,3,6]

	active_designs = []
	completed_designs = []
	platform = []
	sku = []
	mem_type = []
	design_type = []
	temp_board_id = []

	active_designs_temp = []
	completed_designs_temp = []
	platform_temp = []
	sku_temp = []
	mem_type_temp = []
	design_type_temp = []

	sql="SELECT DISTINCT b1.BoardID,b1.BoardName,b1.PlatformID,p1.PlatformName,b1.SKUID,s1.SKUName,b1.MemTypeID,m1.MemTypeName,b1.DesignTypeID,d2.DesignTypeName,s2.ScheduleStatusID FROM BoardDetails b1,Platform p1,SUK s1,MemType m1,DesignType d2,ScheduleTable s2,DesignCalendar d1 WHERE s2.BoardID = b1.BoardID AND p1.PlatformID = b1.PlatformID AND b1.SKUID = s1.SKUID AND b1.MemTypeID = m1.MemTypeID AND b1.DesignTypeID = d2.DesignTypeID AND b1.BoardID = d1.BoardID AND b1.PlatformID IN %s AND b1.SKUID IN %s AND b1.MemTypeID IN %s AND d1.ProposedStartDate >= %s AND d1.ProposedStartDate <= %s ORDER BY b1.BoardID DESC"
	val=(platform_list_input,sku_list_input,mem_type_list_input,start_calender_date,end_calender_date)
	result=execute_query(sql,val)

	if result != ():
		has_data = True

		for i in range(len(result)):
			#designs_temp.append([result[i][0],result[i][1],"checked"])
			if result[i][10] in active_designs_list:
				active_designs_temp.append([result[i][0],result[i][1],"checked"])
			else:
				completed_designs_temp.append([result[i][0],result[i][1],"checked"])

			platform_temp.append([result[i][2],result[i][3],"checked"])
			sku_temp.append([result[i][4],result[i][5],"checked"])
			mem_type_temp.append([result[i][6],result[i][7],"checked"])
			design_type_temp.append([result[i][8],result[i][9],"checked"])

	platform_temp = list(frozenset(tuple(row) for row in platform_temp)) 
	sku_temp = list(frozenset(tuple(row) for row in sku_temp))
	mem_type_temp = list(frozenset(tuple(row) for row in mem_type_temp))
	design_type_temp = list(frozenset(tuple(row) for row in design_type_temp))

	for row in platform_temp:
		platform.append(list(row))

	for row in sku_temp:
		sku.append(list(row))

	for row in mem_type_temp:
		mem_type.append(list(row))

	for row in design_type_temp:
		design_type.append(list(row))

	for row in active_designs_temp:
		active_designs.append(list(row))									

	for row in completed_designs_temp:
		completed_designs.append(list(row))									

	# sorting
	platform.sort(key = lambda x: x[1])
	sku.sort(key = lambda x: x[1])
	mem_type.sort(key = lambda x: x[1])
	design_type.sort(key = lambda x: x[1])

	temp = 0
	for i in platform:
		temp += 1

	platform_all = True if temp > 1 else False

	temp = 0
	for i in sku:
		temp += 1

	sku_all = True if temp > 1 else False

	temp = 0
	for i in mem_type:
		temp += 1

	mem_type_all = True if temp > 1 else False

	temp = 0
	for i in design_type:
		temp += 1

	design_type_all = True if temp > 1 else False


	final_result['platform'] = platform
	final_result['sku'] = sku
	final_result['mem_type'] = mem_type
	final_result['design_type'] = design_type
	final_result['active_designs'] = active_designs
	final_result['completed_designs'] = completed_designs
	
	final_result['platform_all'] = platform_all
	final_result['sku_all'] = sku_all
	final_result['mem_type_all'] = mem_type_all
	final_result['design_type_all'] = design_type_all

	return jsonify(final_result)

@app.route("/get_designs_name_for_summary",methods = ['POST', 'GET'])
def get_designs_name_for_summary():

	array_body = request.form.get("array_body")
	data = json.loads(array_body)

	start_calender_date = data[0]
	end_calender_date = data[1]

	print("start_calender_date: ",start_calender_date)
	print("end_calender_date: ",end_calender_date)

	platform_list_input = data[2]
	sku_list_input = data[3]
	mem_type_list_input = data[4]
	design_type_list_input = data[5]

	final_result = {}

	has_data = False
	active_designs_list = [2,3,6]

	active_designs = []
	completed_designs = []
	platform = []
	sku = []
	mem_type = []
	design_type = []
	temp_board_id = []

	active_designs_temp = []
	completed_designs_temp = []
	platform_temp = []
	sku_temp = []
	mem_type_temp = []
	design_type_temp = []

	sql="SELECT DISTINCT b1.BoardID,b1.BoardName,b1.PlatformID,p1.PlatformName,b1.SKUID,s1.SKUName,b1.MemTypeID,m1.MemTypeName,b1.DesignTypeID,d2.DesignTypeName,s2.ScheduleStatusID FROM BoardDetails b1,Platform p1,SUK s1,MemType m1,DesignType d2,ScheduleTable s2,DesignCalendar d1 WHERE s2.BoardID = b1.BoardID AND p1.PlatformID = b1.PlatformID AND b1.SKUID = s1.SKUID AND b1.MemTypeID = m1.MemTypeID AND b1.DesignTypeID = d2.DesignTypeID AND b1.BoardID = d1.BoardID AND b1.PlatformID IN %s AND b1.SKUID IN %s AND b1.MemTypeID IN %s AND b1.DesignTypeID IN %s AND d1.ProposedStartDate >= %s AND d1.ProposedStartDate <= %s ORDER BY b1.BoardID DESC"
	val=(platform_list_input,sku_list_input,mem_type_list_input,design_type_list_input,start_calender_date,end_calender_date)
	result=execute_query(sql,val)

	if result != ():
		has_data = True

		for i in range(len(result)):
			#designs_temp.append([result[i][0],result[i][1],"checked"])
			if result[i][10] in active_designs_list:
				active_designs_temp.append([result[i][0],result[i][1],"checked"])
			else:
				completed_designs_temp.append([result[i][0],result[i][1],"checked"])

			platform_temp.append([result[i][2],result[i][3],"checked"])
			sku_temp.append([result[i][4],result[i][5],"checked"])
			mem_type_temp.append([result[i][6],result[i][7],"checked"])
			design_type_temp.append([result[i][8],result[i][9],"checked"])

	platform_temp = list(frozenset(tuple(row) for row in platform_temp)) 
	sku_temp = list(frozenset(tuple(row) for row in sku_temp))
	mem_type_temp = list(frozenset(tuple(row) for row in mem_type_temp))
	design_type_temp = list(frozenset(tuple(row) for row in design_type_temp))

	for row in platform_temp:
		platform.append(list(row))

	for row in sku_temp:
		sku.append(list(row))

	for row in mem_type_temp:
		mem_type.append(list(row))

	for row in design_type_temp:
		design_type.append(list(row))

	for row in active_designs_temp:
		active_designs.append(list(row))									

	for row in completed_designs_temp:
		completed_designs.append(list(row))									

	# sorting
	platform.sort(key = lambda x: x[1])
	sku.sort(key = lambda x: x[1])
	mem_type.sort(key = lambda x: x[1])
	design_type.sort(key = lambda x: x[1])

	temp = 0
	for i in platform:
		temp += 1

	platform_all = True if temp > 1 else False

	temp = 0
	for i in sku:
		temp += 1

	sku_all = True if temp > 1 else False

	temp = 0
	for i in mem_type:
		temp += 1

	mem_type_all = True if temp > 1 else False

	temp = 0
	for i in design_type:
		temp += 1

	design_type_all = True if temp > 1 else False


	final_result['platform'] = platform
	final_result['sku'] = sku
	final_result['mem_type'] = mem_type
	final_result['design_type'] = design_type
	final_result['active_designs'] = active_designs
	final_result['completed_designs'] = completed_designs
	
	final_result['platform_all'] = platform_all
	final_result['sku_all'] = sku_all
	final_result['mem_type_all'] = mem_type_all
	final_result['design_type_all'] = design_type_all

	return jsonify(final_result)

@app.route("/page_timeout",methods = ['POST', 'GET'])
def page_timeout():

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	return render('page_timeout.html',username=username,user_role_name=user_role_name,region_name=region_name)

@app.route('/pif_leads', methods=['POST', 'GET'])
def pif_leads():

	wwid =  session.get('wwid')
	username =  session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	board_name = ''
	boardid = 0
	data = []
	error_message = ''
	edit_access = False

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	is_admin = False
	if has_admin_access == "yes":
		is_admin = True
		edit_access = True

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = False
	if(role == 4 or role == 16 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	# only admin, design lead & manager can have edit access
	#if(role == 3 or role == 5):
	#	edit_access = True

	if is_admin:
		pass
	else:
		return render('error_custom.html',error='You do not have access to this page. Please contact admin.',username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "SELECT DISTINCT WWID,UserName,RoleID FROM HomeTable WHERE IsActive = 1 AND RoleID IN (1,12,14) ORDER BY RoleID,UserName"
	user_list = execute_query_sql(sql)

	sql = "SELECT DISTINCT RoleID,RoleName FROM RoleTable WHERE RoleID IN (1,12,14) ORDER BY RoleID"
	role_list = execute_query_sql(sql)

	edit_mode = False

	sql = "SELECT a.PlatformID,a.PlatformName,IFNULL(c.UserName,''),IFNULL(b.SecondaryOwner,''),IFNULL(b.PrimaryOwner,0) FROM Platform a LEFT JOIN PifLeadTable b ON a.PlatformID = b.PlatformID LEFT JOIN HomeTable c ON b.PrimaryOwner = c.WWID ORDER BY a.PlatformName"
	result = execute_query_sql(sql)

	for i in range(len(result)):

		temp_sec_wwid_list = []
		data_temp_list = []
		data_temp_list.append(result[i][0])	# platform ID
		data_temp_list.append(result[i][1])	# platform name
		#data_temp_list.append(result[i][2].replace(',',''))	# primary owner name
		data_temp_list.append(result[i][2])	# primary owner name

		SecondaryOwner = ''
		if(result[i][3] != None):
			rem = result[i][3][1:-1]
			if(rem != None or  rem != "" ):
				spl = rem.split(",")
				if (spl != ['']):
					for j in range(0,len(spl)):
						lead1_wwid = (spl[j])
						if len(lead1_wwid) >= 8:
							
							temp_sec_wwid_list.append(int(lead1_wwid))

							if str(lead1_wwid) != str(99999999):
								sql = "SELECT UserName FROM HomeTable WHERE WWID = %s"
								val=(str(lead1_wwid),)
								name = execute_query(sql,val)

								if name != ():
									if SecondaryOwner == '':
										SecondaryOwner = name[0][0]
									else:
										SecondaryOwner = SecondaryOwner + '<br>' + name[0][0]

		# to remove duplicates wwids
		temp_sec_wwid_list = set(temp_sec_wwid_list)
		temp_sec_wwid_list = list(temp_sec_wwid_list)

		data_temp_list.append(SecondaryOwner)		# sec owner name list for display
		data_temp_list.append(result[i][4])		# primary owner wwid
		data_temp_list.append(temp_sec_wwid_list)	# sec owner wwid list

		data.append(data_temp_list)


	return render("pif_leads.html",is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,username=username,user_role_name=user_role_name,region_name=region_name,edit_mode=edit_mode,edit_access=edit_access,is_admin=is_admin,user_list=user_list,role_list=role_list,data=data)


@app.route("/update_pif_leads",methods = ['POST', 'GET'])
def update_pif_leads():

	wwid=session.get('wwid')
	username = session.get('username')

	max_row_count = request.form.get("max_row_count")

	sql = "SELECT DISTINCT WWID,UserName,RoleID,AdminAccess,EmailID FROM HomeTable WHERE WWID <> 99999999 ORDER BY UserName"
	user_list_rs = execute_query_sql(sql)

	user_list = []
	for row in user_list_rs:
		user_list.append([row[0],row[1],row[2],row[3],row[4]])

	if max_row_count is None:
		max_row_count = 0

	html = '<br><table style="width: 100%;border: 1px solid #ddd;padding: 2px;">'
	html += '<tr><td style="width: 5%;border-bottom: 1px solid #ddd;"><b>S.No</b></td><td style="width: 30%;border-bottom: 1px solid #ddd;"><b>Platform Name</b></td><td style="width: 30%;border-bottom: 1px solid #ddd;"><b>Primary Owner</b></td><td style="width: 35%;border-bottom: 1px solid #ddd;"><b>Secondary Owners</b></td></tr>'

	count = 0

	for i in range(0,int(max_row_count)):
		platform_id = request.form.get("platform_id_"+str(i))
		primary_owner = request.form.get("primary_owner_"+str(i))
		sec_owner = request.form.getlist("sec_owner_"+str(i))

		is_valid_update = False

		if primary_owner is not None:
			is_valid_update = True

		if is_valid_update:

			sec_owner = [int(x) for x in sec_owner]

			if sec_owner == []:
				sec_owner = [99999999]

			if primary_owner is not None:
				sql="SELECT PlatformName FROM Platform WHERE PlatformID = %s"
				val =(platform_id,)
				rs_comp=execute_query(sql,val)

				plat_name = ''
				if rs_comp != ():
					plat_name = rs_comp[0][0]

				# log table
				try:
					log_notes = 'User has updated Package Owner details for Platform: '+str(plat_name)
					log_notes += '<br><br>Primary Owner: '+str(primary_owner)+'<br>Secondary Owner: '+str(sec_owner)
					log_wwid = session.get('wwid')
					t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
					sql = "INSERT INTO LogTable(Category,BoardID,RequestID,ComponentID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s,%s)"
					val = ('PIF Leads',0,0,0,log_wwid,t,log_notes)
					execute_query(sql,val)
				except:
					if is_logging:
						logging.exception('')
					print("log error.")

				sql = "INSERT INTO PifLeadTable(PlatformID,PrimaryOwner,SecondaryOwner) VALUES (%s,%s,%s) ON DUPLICATE KEY UPDATE PrimaryOwner = %s, SecondaryOwner = %s"
				val = (platform_id,primary_owner,str(sec_owner),primary_owner,str(sec_owner))
				execute_query(sql,val)

				count += 1
				primary_owner_name = ''
				sec_owner_name = ''
				for row in user_list:
					if row[0] == int(primary_owner):
						primary_owner_name = copy.deepcopy(row[1])

					if row[0] in sec_owner:
						if sec_owner_name == '':
							sec_owner_name = copy.deepcopy(row[1])
						else:
							sec_owner_name += '<br>'+row[1]



				html += '<tr><td style="border-bottom: 1px solid #ddd;">'+str(count)+'</td><td style="border-bottom: 1px solid #ddd;">'+str(plat_name)+'</td><td style="border-bottom: 1px solid #ddd;">'+str(primary_owner_name)+'</td><td style="border-bottom: 1px solid #ddd;">'+str(sec_owner_name)+'</td></tr>'

	html += '</table>'

	if count > 0:
		subject = "PIF Leads - Updated"
		message = 'Hi,<br><br>PIF Lead details has been updated by <b>'+username+'</b> as below,<br><br>'
		message += html
		email_list = []

		for row in user_list:

			# all admin users
			if row[3] == "yes":
				email_list.append(row[4])

			# who updated
			if row[0] == wwid:
				email_list.append(row[4])

		email_list = sorted(set(email_list), reverse=True)
		email_list = list(email_list)
		
		for i in email_list:
			send_mail_html(i,subject,message,email_list)

	return redirect(url_for('pif_leads', _external=True))

def get_pif_leads_email_id_by_platform_id(platformid=0):

	data = []

	sql = "SELECT DISTINCT WWID,UserName,RoleID,EmailID FROM HomeTable ORDER BY RoleID,UserName"
	user_list = execute_query_sql(sql)

	sql = "SELECT IFNULL(c.EmailID,''),IFNULL(b.SecondaryOwner,'') FROM Platform a LEFT JOIN PifLeadTable b ON a.PlatformID = b.PlatformID LEFT JOIN HomeTable c ON b.PrimaryOwner = c.WWID WHERE a.PlatformID = %s"
	val = (platformid,)
	result = execute_query(sql,val)

	# pif lead primary owner
	if result != ():
		if result[0][0] != '':
			data.append(result[0][0])

	# pif lead secondary owner
	for i in range(len(result)):

		if(result[i][1] != None):
			rem = str(result[i][1])[1:-1]
			if(rem != None or  rem != "" ):
				spl = rem.split(", ")
				if type(spl) == list:
					for row in user_list:
						if str(row[0]) in spl:
							data.append(row[3])

	#print(data)

	return data


def get_pif_leads_email_id_by_board_id(boardid=0):

	data = []

	sql = "SELECT DISTINCT WWID,UserName,RoleID,EmailID FROM HomeTable ORDER BY RoleID,UserName"
	user_list = execute_query_sql(sql)

	sql = "SELECT IFNULL(c.EmailID,''),IFNULL(b.SecondaryOwner,'') FROM Platform a LEFT JOIN BoardDetails bd ON a.PlatformID = bd.PlatformID LEFT JOIN PifLeadTable b ON a.PlatformID = b.PlatformID LEFT JOIN HomeTable c ON b.PrimaryOwner = c.WWID WHERE bd.BoardID = %s"
	val = (boardid,)
	result = execute_query(sql,val)

	# pif lead primary owner
	if result != ():
		if result[0][0] != '':
			data.append(result[0][0])

	# pif lead secondary owner
	for i in range(len(result)):

		if(result[i][1] != None):
			rem = str(result[i][1])[1:-1]
			if(rem != None or  rem != "" ):
				spl = rem.split(", ")
				if type(spl) == list:
					for row in user_list:
						if str(row[0]) in spl:
							data.append(row[3])

	#print(data)

	return data

@app.route("/get_pif_lead_name",methods = ['POST', 'GET'])
def get_pif_lead_name():
	data = json.loads(request.form.get("data"))

	data[0] = data[0].replace(" ","+")

	sql="SELECT PlatformID FROM Platform WHERE PlatformName = %s"
	val=(data[0],)
	result_rs=execute_query(sql,val)

	platform_id = 0
	if result_rs != ():
		platform_id = result_rs[0][0]

	sql="SELECT IFNULL(b.UserName,'') FROM PifLeadTable a LEFT JOIN HomeTable b ON a.PrimaryOwner = b.WWID WHERE b.WWID <> 99999999 AND a.PlatformID = %s"
	val=(platform_id,)
	result=execute_query(sql,val)

	if result != ():
		pif_name = result[0][0]
	else:
		pif_name = ''

	return jsonify(pif_name)


# automation #1 - updating design status to 'Projected and No updates' for below conditions,
# a.	If design files are not uploaded and design status is ERAM timeline commit or Design Team Projection
# b.	Design start date < current date - change the design status to 'Projected and No updates' (trigger email and use the same email/recipient as of update design timelines)
def check_design_status():
	
	print("Automation #1 - Design status update")
	date_today = datetime.datetime.now(tz).date()

	try:
		boardstate = "Projected & No-Updates"

		sql = "SELECT a.BoardID FROM BoardDetails a,DesignCalendar b WHERE a.BoardID = b.BoardID AND a.BoardID NOT IN (SELECT c.BoardID FROM UploadDesignFiles c) AND b.BoardState IN ('ERAM Timeline Commit','Design Team Projection') AND b.ProposedStartDate < %s"
		val = (date_today,)
		result=execute_query(sql,val)

		for i in range(len(result)):
			boardid = copy.deepcopy(result[i][0])
			print("updating for boardid: ",boardid)

			# update calender status
			sql = "UPDATE DesignCalendar SET BoardState = %s WHERE BoardID = %s"
			val = (boardstate,boardid)
			result_temp = execute_query(sql,val)

			# trigger email
			sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
			val = (boardid,)
			boardname = execute_query(sql,val)[0][0]

			email_list = []
			query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
			val = (boardid,)
			designlist = execute_query(query,val)
			designlead_list = []
			for i in range(len(designlist)):
				eid = designlist[0][1]
				email_list.append(eid)


			query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
			val = (boardid,)
			cadlist = execute_query(query,val)
			cadlead_list = []
			for i in range(len(cadlist)):
				eid = cadlist[0][1]
				email_list.append(eid)


			query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.PIFLeadWWID and b.BoardID AND b.BoardID = %s)"
			piflist = execute_query(query,val)
			piflead_list = []
			for i in range(len(piflist)):
				eid = piflist[0][1]
				email_list.append(eid)

			# all pif leads
			email_list += get_pif_leads_email_id_by_board_id(boardid=boardid)

			sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
			val=('yes',)
			admin_list = execute_query(sql,val)				

			for j in admin_list:
				email_list.append(j[0])


			subject =  "[ID: "+str(boardid)+"] Details Updated"
			message = '''Please find the updated details for <b>'''+boardname+'''</b><br>'''

			message += '''<br><u>Design State: ''' + boardstate + '''</u>'''

			message += '''<br><br>Regards,<br>ERAM.'''

			if boardstate not in ('Design Signed-off','Design Not signed-off','Design Review In-Progress'):

				email_list = sorted(set(email_list), reverse=True)
				for i in email_list:
					print("sending mail now..")
					send_mail(i,subject,message,email_list)

	except Exception as inst:
		if is_logging:
			logging.exception('')
		print("Error in updating automation #1 - ",inst)

	return True

# automation #2 - On design files upload for yet to kickstart design - one time activity
#i.	Projected & No updates : change the start date as current date and update end date accordingly. No email trigger.  Change status to Design Team Projection
def updte_design_status_and_dates():

	print("Automation #2 - change start and end date")

	try:
		boardstate = "Projected & No-Updates"
		new_boardstate = "Design Team Projection"

		#sql = "SELECT a.BoardID,a.BoardTrackID,b.ProposedStartDate,b.ProposedEndDate FROM BoardDetails a,DesignCalendar b WHERE a.BoardID = b.BoardID AND a.BoardID IN (SELECT c.BoardID FROM UploadDesignFiles c) AND b.BoardState = %s"
		sql = "SELECT a.BoardID,a.BoardTrackID,b.ProposedStartDate,b.ProposedEndDate FROM BoardDetails a,DesignCalendar b WHERE a.BoardID = b.BoardID AND a.BoardID IN (SELECT DISTINCT c.BoardID FROM BoardReviewDesigner c WHERE c.IsPdgDesignSubmitted = %s) AND b.BoardState = %s"
		val = ("yes",boardstate,)
		result=execute_query(sql,val)

		for i in range(len(result)):
			boardid = copy.deepcopy(result[i][0])

			start_date = datetime.datetime.now(tz).date()

			no_of_days = get_no_of_working_days(start_date=result[i][2],end_date=result[i][3])
			end_date = get_work_week_addition(date_value=start_date,no_of_days=no_of_days)

			# update calender status
			sql = "UPDATE DesignCalendar SET BoardState = %s, ProposedStartDate = %s, ProposedEndDate = %s WHERE BoardID = %s"
			val = (new_boardstate,start_date,end_date,boardid)
			result_temp = execute_query(sql,val)

	except Exception as inst:
		if is_logging:
			logging.exception('')
		print("Error in updating automation #2 - ",inst)

	return True

def get_no_of_working_days(start_date,end_date):

	no_of_days = 0

	date_diff = (end_date - start_date).days
	start_date_temp = copy.deepcopy(start_date)

	for i in range(0,date_diff):
		start_date_temp = start_date + datetime.timedelta(days=i)
		if str(get_isocalendar(start_date_temp)[2]) not in ['6','7']:
			no_of_days += 1

	return no_of_days

def reminder_mail(boardid=0):

	try:

		date = datetime.datetime.now(tz).strftime('%Y-%m-%d')

		sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
		val = (boardid,)
		try:
			boardname = execute_query(sql,val)[0][0]
		except:
			boardname = ''

		sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
		val = (boardid,)
		cal = execute_query(sql,val)
		
		try:
			d1 = get_work_week_date_fmt(cal[0][1])
			d2 = get_work_week_date_fmt(cal[0][2])
		except:
			d1 = ''
			d2 = ''

		sql = "SELECT ScheduleStatusID FROM ScheduleTable WHERE BoardID = %s"
		val = (boardid,)
		try:
			state = execute_query(sql,val)[0][0]
		except:
			state = 2

		wwid = session.get('wwid')

		sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
		val = (boardid,)
		try:
			rid = execute_query(sql,val)[0][0]
		except:
			rid = 1

		timeline = ""
		if(rid == 1):
			timeline = "Core/Placement layout review Rev0p6"
		else:
			timeline = "Full layout review Rev1p0"

		subject = ''
		message = ''

		if(state == 2):
			status_interface = ""	

			subject= "[ID:"+str(boardid)+"] Reminder for review request. Please Provide Feedback By " + d2
			message=''' Gentle Reminder, <br><br>

			This is a '''+timeline+''' request for ''' + boardname + ''' <br>

			Please provide the feedback / sign-off by <font style="color:red">''' + d2 +'''</font><br>  
			

			Please proceed to visit https://eram.apps1-fm-int.icloud.intel.com/ to submit your feedback/signoff. <br><br>'''


		elif(state == 3):

			subject= "[ID: "+str(boardid)+"] Reminder for review request. Please Update Details By " + d1
			message=''' Gentle Reminder, <br><br>

			Start Date of '''+timeline+''' request for <b>''' + boardname + '''</b> is <b>'''+ d1 +'''</b> <br><br>

			<u><b> AR To Design/Layout Lead </b></u> <br>
			Please proceed to https://eram.apps1-fm-int.icloud.intel.com/ to submit feedback/sign-off. <br>
			To provide Feedback/Sign-Off : My Dashboard >> Feedback Submission Module >> '''+boardname+'''<br><br>'''


		sql = "select UserName,WWID from HomeTable ORDER BY UserName"
		usernames = execute_query_sql(sql)
		user_list = []
		for i in usernames:
			user_list.append(i)

		email_list = []
		user_specific_comp_name = []

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
		val = (boardid,)
		designlist = execute_query(query,val)
		designlead_list = []
		for i in range(len(designlist)):
			eid = designlist[0][1]
			email_list.append(eid)

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.Username=b.DesignManagerWWID and b.BoardID AND b.BoardID = %s) LIMIT 1"
		val = (boardid,)
		designmanager = execute_query(query,val)
		if designmanager != ():
			email_list.append(designmanager[0][1])

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
		val = (boardid,)
		cadlist = execute_query(query,val)
		cadlead_list = []
		for i in range(len(cadlist)):
			eid = cadlist[0][1]
			email_list.append(eid)


		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.PIFLeadWWID and b.BoardID AND b.BoardID = %s)"
		piflist = execute_query(query,val)
		piflead_list = []
		for i in range(len(piflist)):
			eid = piflist[0][1]
			email_list.append(eid)

		# all pif leads
		email_list += get_pif_leads_email_id_by_board_id(boardid=boardid)

		sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
		val=('yes',)
		admin_list = execute_query(sql,val)				

		for j in admin_list:
			email_list.append(j[0])


		sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
		val = (boardid,)
		sku_plat = execute_query(sql,val)

		sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
		val = (boardid,)
		cal = execute_query(sql,val)

		sql="SELECT a.CategoryName,b.EmailID,C3.CategoryLeadWWID1 from ComponentReview cr, CategoryType a, HomeTable b, CategoryLeadTable C3,ComponentType C2,BoardReviewDesigner B1,ScheduleTableComponent S1 WHERE cr.SKUID = %s AND cr.PlatformID = %s AND cr.MemTypeID = %s AND cr.DesignTypeID = %s AND cr.ComponentID = C2.ComponentID and C2.CategoryID = a.CategoryID AND a.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID = %s AND C3.CategoryLeadWWID = b.WWID AND C2.ComponentID = B1.ComponentID AND B1.BoardID = S1.BoardID AND B1.ComponentID = S1.ComponentID AND B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND S1.ScheduleStatusID IN (2,3,6,NULL) ORDER BY cr.ComponentID"
		val=(sku_plat[0][0], sku_plat[0][1],sku_plat[0][2], sku_plat[0][3],sku_plat[0][0], sku_plat[0][1],sku_plat[0][2], sku_plat[0][3],boardid,"yes")
		catlead=execute_query(sql,val)
		if(catlead != ()):
			for i in catlead:
				email_list.append(i[1])		

				if i[2] is not None:
					if i[2] != []:
						cat_sec_wwid_list = i[2][1:-1].split(', ')

						for j in range(0,len(cat_sec_wwid_list)):
							like_user_wwid = '%' + str(cat_sec_wwid_list[j]) + '%'
							if cat_sec_wwid_list[j] not in ['99999999',99999999]:
								sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
								val=(like_user_wwid,)
								catlead_sec_mail_rs = execute_query(sql,val)

								if catlead_sec_mail_rs != ():
									email_list.append(catlead_sec_mail_rs[0][0])

		if(state == 2):
			sql = "SELECT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1,ComponentType C1,ScheduleTableComponent S1, ScheduleStatusType S2 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND S1.ScheduleStatusID IN (2,3,6,NULL) ORDER BY FIELD(S1.ScheduleStatusID, 3, 6, 2, 1, 4, 5, 7),C1.ComponentName"
			val = (boardid,"yes")
			compids = execute_query(sql,val)
			
			status = ""
			status_interface = '<b><u>Interface list in Yet to Kickstart and Ongoing state: </u></b><br><br><table style="width: 100%;border: 1px solid #ddd;padding: 2px;">'
			status_interface += '<tr><td style="width: 35%;border-bottom: 1px solid #ddd;"><b>Interface</b></td><td style="width: 20%;border-bottom: 1px solid #ddd;"><b>Status</b></td><td style="width: 45%;border-bottom: 1px solid #ddd;"><b>Primary Electrical Owner</b></td></tr>'

			for q in compids:
				
				sql = "SELECT SecondaryWWID from ComponentReview C2 WHERE  C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s"
				val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],)
				sec_ele_wwid = execute_query(sql,val)

				sec_wwid = []
				if sec_ele_wwid != ():
					for i in sec_ele_wwid:
						if i[0] is not None:
							if (i[0] != []) and (i[0] != ['']):
								sec_wwid = i[0][1:-1].split(', ')

								for j in range(0,len(sec_wwid)):
									like_user_wwid = '%' + str(sec_wwid[j]) + '%'
									if sec_wwid[j] not in ['99999999',99999999]:
										sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
										val=(like_user_wwid,)
										email_id_rs = execute_query(sql,val)

										if email_id_rs != ():
											email_list.append(email_id_rs[0][0])

				sql = "select SecondaryWWID from ComponentDesign C2 WHERE  C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s"
				val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],)
				sec_des_wwid = execute_query(sql,val)

				des_wwid = []
				if sec_des_wwid != ():
					for i in sec_des_wwid:
						if i[0] is not None:
							if (i[0] != []) and (i[0] != ['']):
								des_wwid = i[0][1:-1].split(', ')

								for j in range(0,len(des_wwid)):
									like_user_wwid = '%' + str(des_wwid[j]) + '%'
									if des_wwid[j] not in ['99999999',99999999]:
										sql = "SELECT EmailID FROM HomeTable WHERE WWID like %s"
										val=(like_user_wwid,)
										email_id_rs = execute_query(sql,val)

										if email_id_rs != ():
											email_list.append(email_id_rs[0][0])


				sql = "SELECT H1.EmailID FROM HomeTable H1,ComponentDesign C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID"
				val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],)
				primary_rev = execute_query(sql,val)

				sql = "SELECT H1.EmailID,IFNULL(H1.UserName,'') FROM HomeTable H1,ComponentReview C2 WHERE C2.ComponentID = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND  C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C2.PrimaryWWID = H1.WWID "
				val = (q[0],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],)
				primary_des = execute_query(sql,val)

				font_color = ''
				if q[3] == 1:	# signed-off
					font_color = 'green'
				elif q[3] == 2:	# ongoing
					font_color = 'orange'
				elif q[3] == 3:	# yet to kickstart
					font_color = 'red'

				if primary_des != ():
					status_interface += '<tr><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">'+q[1] + '</td><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">' + q[2] + '</td><td style="border-bottom: 1px solid #ddd;">' + primary_des[0][1] + '</td></tr>'

					if q[3] not in [1,'1',7,'7']:
						user_specific_comp_name.append([str(primary_des[0][0]),str(primary_des[0][1]),str(q[1])])

				else:
					status_interface += '<tr><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">'+q[1] + '</td><td style="border-bottom: 1px solid #ddd;color:'+font_color+'">' + q[2] + '</td><td style="border-bottom: 1px solid #ddd;">-</td></tr>'

				sql = "SELECT H.EmailID FROM HomeTable H WHERE H.RoleID = 14"
				mgmt = execute_query_sql(sql)
					
				for j in primary_rev:
					email_list.append(j[0])

				for j in primary_des:
					email_list.append(j[0])

				for j in mgmt:
					for k in range(len(j)):
						email_list.append(j[k])
				status = status + q[1] + "&nbsp &nbsp &nbsp" + q[2] + "<br>"	

					
			d1 = str(cal[0][2])	
			email_list = sorted(set(email_list), reverse=True)

			status_interface += "</table>"

			for i in email_list:
				
				x=0
				temp_user_specific_msg = ''
				message_temp = ''

				for j in range(len(user_specific_comp_name)):
					if user_specific_comp_name[j][0] == i:
						if x==0:
							temp_user_specific_msg += '<font style="background-color:yellow;"><u>Interfaces to be reviewed/Signed-off by <b>'+user_specific_comp_name[j][1]+':</b></u></font><br>'
							temp_user_specific_msg += user_specific_comp_name[j][2]
						else:
							temp_user_specific_msg += '<br>'+user_specific_comp_name[j][2]
						x+=1

				message_temp += message + temp_user_specific_msg + '<br><br>'
				message_temp += status_interface
				message_temp += '<br><br>Thanks,<br>ERAM.'

				send_mail_html(i,subject,message_temp,email_list)	

		elif(state == 3):

			d1 = str(cal[0][1])	

			message += '<br><br>Thanks,<br>ERAM.'

			email_list = sorted(set(email_list), reverse=True)
			for i in email_list:
				send_mail(i,subject,message,email_list)

	except Exception as inst:
		if is_logging:
			logging.exception('')
		print("Error in updating reminder automation - ",inst)

	return True

def reminder_automation():

	print("Automation #3 - Reminder mail")

	tz = pytz.timezone('Asia/Kolkata')
	date_ist = datetime.datetime.now(tz).strftime('%Y-%m-%d')

	trigger_time = datetime.datetime.now(tz).replace(hour=6, minute=0, second=0, microsecond=0)
	current_time = datetime.datetime.now(tz)
	
	if current_time > trigger_time:
		#print("yes")

		sql = "SELECT a.BoardID FROM BoardDetails a,ScheduleTable b,DesignCalendar c WHERE a.BoardID = b.BoardID AND a.BoardID = c.BoardID AND b.ScheduleStatusID = %s AND c.ProposedEndDate = %s AND a.BoardID NOT IN (SELECT aa.BoardID FROM ReminderAutomation aa)"
		val = (2,date_ist)
		result = execute_query(sql,val)

		for row in result:
			boardid = copy.deepcopy(row[0])

			reminder_mail(boardid=boardid)
			
			sql = "INSERT INTO ReminderAutomation(BoardID) VALUES(%s)"
			val = (boardid,)
			result_update = execute_query(sql,val)

	return True

@app.route("/discard_edit_feedbacks_design_files",methods = ['POST', 'GET'])
def discard_edit_feedbacks_design_files():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	board_id = data["board_id"]
	comp_id = data["comp_id"]
	comment_id = data["comment_id"]
	comp_selected_list = data["comp_select"]
	#comp_selected_list = eval(request.form.get("comp_select[]"))

	#data =[]

	sql = "SELECT * FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
	val = (board_id,comp_id,comment_id)
	rs_temp = execute_query(sql,val)

	if rs_temp != ():
		sql = "DELETE FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
		val = (board_id,comp_id,comment_id)
		execute_query(sql,val)

		sql = "INSERT INTO BoardReview SELECT * FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
		val = (board_id,comp_id,comment_id)
		execute_query(sql,val)

		sql = "UPDATE BoardReview SET EditDiscardFlag = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
		val = (0,board_id,comp_id,comment_id)
		execute_query(sql,val)

		sql = "DELETE FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
		val = (board_id,comp_id,comment_id)
		execute_query(sql,val)

	#return design_files(boardid=int(board_id),comp_selected_list=comp_selected_list,my_designs="All",my_interfaces="All")
	return jsonify(True)

@app.route("/discard_edit_feedbacks",methods = ['POST', 'GET'])
def discard_edit_feedbacks():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))
	
	board_id = data["board_id"]
	comp_id = data["comp_id"]
	comment_id = data["comment_id"]
	comp_selected_list = data["comp_select"]

	#data =[]

	sql = "SELECT * FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
	val = (board_id,comp_id,comment_id)
	rs_temp = execute_query(sql,val)

	if rs_temp != ():
		sql = "DELETE FROM BoardReview WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
		val = (board_id,comp_id,comment_id)
		execute_query(sql,val)

		sql = "INSERT INTO BoardReview SELECT * FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
		val = (board_id,comp_id,comment_id)
		execute_query(sql,val)

		sql = "UPDATE BoardReview SET EditDiscardFlag = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
		val = (0,board_id,comp_id,comment_id)
		execute_query(sql,val)

		sql = "DELETE FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
		val = (board_id,comp_id,comment_id)
		execute_query(sql,val)

	return jsonify(True)

def reminder_automation_upload_files():

	print("Automation #4 - Reminder mail for uploading files.")

	date_today = datetime.datetime.now(tz).date()

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	design_status = ["Design Team Projection","ERAM Timeline Commit"]

	# Design is in yet to kickstart state ('Design Team Projection' and 'ERAM Timeline commit') and current date = start date - 2days(excluding weekends):
	#	a.	Reminder to design team on upcoming design to upload the files or update the design timelines. (ignore point b for first trigger)

	sql = "SELECT a.BoardID,c.ProposedStartDate FROM BoardDetails a, DesignCalendar c WHERE a.BoardID = c.BoardID AND c.BoardState IN %s AND a.BoardID NOT IN (SELECT aa.BoardID FROM ReminderAutomationUploadFiles aa)"
	val = (design_status,)
	result = execute_query(sql,val)
	#print(result)
	#print(",,,,,,,,,,")

	for i in range(0,len(result)):

		is_valid = False

		boardid = result[i][0]

		if result[i][1] == get_work_week_addition(date_value=date_today,no_of_days=2):
			#print("valid first trigger")
			is_valid = True

		if is_valid:
			try:
				sql = "INSERT INTO ReminderAutomationUploadFiles (BoardID,TriggeredOn) VALUES (%s,%s)"
				val = (result[i][0],t)
				rs = execute_query(sql,val)
			except:
				if is_logging:
					logging.exception('')
				print("Insert error")

			# log table
			try:
				log_notes = 'Automation has triggered reminder mail for Design ID: '+str(boardid)
				log_wwid = 99999999
				t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
				sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
				val = ('Reminder',boardid,0,log_wwid,t,log_notes)
				execute_query(sql,val)
			except:
				if is_logging:
					logging.exception('')
				print("log error.")

			# for triggering reminder mail for PIF leads
			reminder_automation_pif_lead(boardid=result[i][0])

			# for triggerring reminder email for Design, layout leads
			reminder_automation_design_lead(boardid=result[i][0])

	# b.	Trigger multiple reminder emails until design is in 'Design Team Projection' and 'ERAM Timeline commit' and last request update > 3 days before review start date.
	sql = "SELECT a.BoardID,c.ProposedStartDate,a.UpdatedOn FROM BoardDetails a, DesignCalendar c WHERE a.BoardID = c.BoardID AND c.BoardState IN %s AND a.BoardID NOT IN (SELECT aa.BoardID FROM ReminderAutomationUploadFiles aa WHERE DATE(aa.TriggeredOn) = %s)"
	val = (design_status,date_today)
	result = execute_query(sql,val)
	#print(result)
	#print(";;;;;;;;;;;;;;;;;;;;;;;;;")

	for i in range(0,len(result)):

		is_valid = False

		boardid = result[i][0]

		if result[i][2]:
			if result[i][2] == get_work_week_addition(date_value=result[i][1],no_of_days=3):
				print("valid multiple trigger")
				is_valid = True

		if is_valid:
			try:
				sql = "INSERT INTO ReminderAutomationUploadFiles (BoardID,TriggeredOn) VALUES (%s,%s)"
				val = (result[i][0],t)
				rs = execute_query(sql,val)
			except:
				if is_logging:
					logging.exception('')
				print("Insert error")

			# log table
			try:
				log_notes = 'Automation has triggered reminder mail for Design ID: '+str(boardid)
				log_wwid = 99999999
				t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
				sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
				val = ('Reminder',boardid,0,log_wwid,t,log_notes)
				execute_query(sql,val)
			except:
				if is_logging:
					logging.exception('')
				print("log error.")

			# for triggering reminder mail for PIF leads
			reminder_automation_pif_lead(boardid=result[i][0])

			# for triggerring reminder email for Design, layout leads
			reminder_automation_design_lead(boardid=result[i][0])

	return True

def reminder_automation_pif_lead(boardid=0):

	date_today = datetime.datetime.now(tz).date()
	date_today_ww = get_work_week_date_fmt(date_today)

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	try:
		boardname = execute_query(sql,val)[0][0]
	except:
		boardname = ''

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)
	
	try:
		d1 = get_work_week_date_fmt(cal[0][1])
		d2 = get_work_week_date_fmt(cal[0][2])
	except:
		d1 = ''
		d2 = ''

	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	try:
		rid = execute_query(sql,val)[0][0]
	except:
		rid = 1

	timeline = ""
	if(rid == 1):
		timeline = "Core/Placement layout review Rev0p6"
	else:
		timeline = "Full layout review Rev1p0"

	email_list = []

	# all pif leads
	email_list += get_pif_leads_email_id_by_board_id(boardid=boardid)

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val=('yes',)
	admin_list = execute_query(sql,val)				

	for j in admin_list:
		email_list.append(j[0])

	subject = "[ID: "+str(boardid)+"] Reminder to update Electrical Owners details for review request."
	message = '''Gentle Reminder,<br><br>
	Start Date of '''+timeline+''' request for '''+boardname+''' is <span style="font-weight:bold;color:red;">'''+str(d1)+'''</span><br>
	Please update the electrical owner details in ERAM.<br>
	In case of new design either import from previous designs or Add interfaces. Please reach out to Admin in case of any new interface addition.<br><br>
	<b><u>AR To PIF Lead:</u></b><br>
	Please proceed to https://eram.apps1-fm-int.icloud.intel.com/ to update details.<br>
	<u>To Update details:</u> My Dashboard >> Add/Update Electrical Owner >> ID: '''+str(boardid)+''' - '''+boardname+'''
	<br><br>Thanks,<br>ERAM.
	'''

	email_list = sorted(set(email_list), reverse=True)
	for i in email_list:
		send_mail_html(i,subject,message,email_list)

	return True

def reminder_automation_design_lead(boardid=0):

	date_today = datetime.datetime.now(tz).date()
	date_today_ww = get_work_week_date_fmt(date_today)

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	try:
		boardname = execute_query(sql,val)[0][0]
	except:
		boardname = ''

	sql = "SELECT RequestID FROM RequestMap WHERE BoardID = %s"
	val = (boardid,)
	try:
		request_id = execute_query(sql,val)[0][0]
	except:
		request_id = ''

	sql = "SELECT * FROM DesignCalendar WHERE BoardID = %s"
	val = (boardid,)
	cal = execute_query(sql,val)
	
	try:
		d1 = get_work_week_date_fmt(cal[0][1])
		d2 = get_work_week_date_fmt(cal[0][2])
	except:
		d1 = ''
		d2 = ''

	sql = "SELECT ReviewTimelineID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	try:
		rid = execute_query(sql,val)[0][0]
	except:
		rid = 1

	timeline = ""
	if(rid == 1):
		timeline = "Core/Placement layout review Rev0p6"
	else:
		timeline = "Full layout review Rev1p0"

	email_list = []

	sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
	val=('yes',)
	admin_list = execute_query(sql,val)				

	for j in admin_list:
		email_list.append(j[0])

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
	val = (boardid,)
	designlist = execute_query(query,val)
	designlead_list = []
	for i in range(len(designlist)):
		eid = designlist[0][1]
		email_list.append(eid)

	'''
	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.Username=b.DesignManagerWWID and b.BoardID AND b.BoardID = %s) LIMIT 1"
	val = (boardid,)
	designmanager = execute_query(query,val)
	if designmanager != ():
		email_list.append(designmanager[0][1])
	'''

	query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
	val = (boardid,)
	cadlist = execute_query(query,val)
	cadlead_list = []
	for i in range(len(cadlist)):
		eid = cadlist[0][1]
		email_list.append(eid)

	subject = "[ID: "+str(boardid)+"] Reminder to upload collaterals for review request."
	message = '''Gentle Reminder,<br><br>
	Start Date of '''+timeline+''' request for '''+boardname+''' is <span style="font-weight:bold;color:red;">'''+str(d1)+'''</span><br>
	Please update the PDG/WP compliance details and upload the required collaterals in ERAM. Please update the board owner's details.<br>
	Please update<br>
	&nbsp;&nbsp;&nbsp;&nbsp;1. PDG/WP compliance details and upload the required collaterals.<br>
	&nbsp;&nbsp;&nbsp;&nbsp;2. Board owner's details.<br>
	&nbsp;&nbsp;&nbsp;&nbsp;3. Timeline in case of change in schedule.<br><br>


	<b><u>AR To Design/Layout Lead:</u></b><br>
	Please proceed to https://eram.apps1-fm-int.icloud.intel.com/ to update details.<br>
	<u>To Update details:</u> My Dashboard >> Design File Submission Module >>  ID: '''+str(boardid)+''' - '''+boardname+''' >> Select Interface Applicable for review and update required details.<br>
	<u>To Update Board owners:</u> My Dashboard >> Add/Update Design Owners >> ID: '''+str(boardid)+''' - '''+boardname+'''.<br>
	<u>To Update timelines:</u> Design Review Projection >> My Design Request >> Request ID : '''+str(request_id)+''' >> Update Request.

	<br><br>Thanks,<br>ERAM.
	'''

	email_list = sorted(set(email_list), reverse=True)
	for i in email_list:
		send_mail_html(i,subject,message,email_list)

	return True

def get_order_status_list(list=[]):

	list_order = ['Yet_to_Kickstart','Ongoing','Reopened','Signed-Off','Rejected','No_Signoff','Signoff_Overdue']
	list_temp = []

	for row in list_order:
		if row in list:
			list_temp.append(row)

	return list_temp

@app.route('/users', methods=['POST', 'GET'])
def users():

	if session.get('wwid') is None:
		print("invalid sso login")
		session['target_page'] = 'review_request'
		sso_url = url_for('sso', _external=True)
		windows_auth_url = build_endpoint_url(API_BASE_URL, 'v1', 'windows/auth')
		redirect_url = windows_auth_url + '?redirecturl=' + sso_url

		return redirect(redirect_url)

	wwid = session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	#if not is_admin:
	#	return render('error_custom.html',error='You do not have access to this page. Please contact admin to get access.',username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "SELECT a.UserName,a.WWID,b.RoleName,a.EmailID,IF(a.IsActive=0,'In-active','Active') FROM HomeTable a,RoleTable b WHERE a.RoleID = b.RoleID ORDER BY a.IsActive DESC,b.RoleName,a.UserName"
	result = execute_query_sql(sql)

	return render("users.html",is_admin=is_admin,username=username,user_role_name=user_role_name,region_name=region_name,request=result)

@app.route("/update_user_status",methods = ['POST', 'GET'])
def update_user_status():

	user_wwid = request.form.get("user_wwid")
	user_status = request.form.get("user_status")

	wwid = session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	if not is_admin:
		return render('error_custom.html',error='You do not have access.',username=username,user_role_name=user_role_name,region_name=region_name)

	if user_status in [1,"1"]:
		log_user_status = "Active"
	else:
		log_user_status = "In-active"

	# log table
	try:
		log_notes = 'User Status changed to '+str(log_user_status)+' for WWID: '+str(user_wwid)+' by Admin ('+str(username)+').'
		log_wwid = user_wwid
		t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
		sql = "INSERT INTO LogTable(Category,BoardID,RequestID,WWID,LogTime,Comments) VALUES (%s,%s,%s,%s,%s,%s)"
		val = ('User Status',0,0,log_wwid,t,log_notes)
		execute_query(sql,val)
	except:
		if is_logging:
			logging.exception('')
		print("log error.")

	sql = "UPDATE HomeTable SET IsActive = %s WHERE WWID = %s"
	val = (user_status,user_wwid)
	execute_query(sql,val)

	sql = "SELECT IsActive FROM HomeTable WHERE WWID = %s"
	val = (user_wwid,)
	result_rs = execute_query(sql,val)

	result = ""

	if result_rs != ():
		if result_rs[0][0] == 1:
			result = "Active"
		else:
			result = "In-active"

	return jsonify(result)

def remove_invalid_interfaces(boardid=0):
	print("remove_invalid_interfaces:")

	sql = "SELECT a.BoardID,a.ComponentID FROM ScheduleTableComponent a WHERE a.BoardID = %s AND a.ScheduleStatusID = %s AND a.ComponentID NOT IN (SELECT b.ComponentID FROM BoardReviewDesigner b WHERE a.BoardID=b.BoardID) ORDER BY a.ComponentID"
	val = (boardid,3)
	result_rs = execute_query(sql,val)
	print("result_rs: ",result_rs)

	for row in result_rs:
		print("row: ",row)
		sql = "DELETE FROM ScheduleTableComponent WHERE BoardID = %s AND ComponentID = %s AND ScheduleStatusID = %s"
		val = (row[0],row[1],3)
		execute_query(sql,val)

		sql = "DELETE FROM BoardReview WHERE BoardID = %s AND ComponentID = %s"
		val = (row[0],row[1])
		execute_query(sql,val)

		sql = "DELETE FROM BoardReviewTemp WHERE BoardID = %s AND ComponentID = %s"
		val = (row[0],row[1])
		execute_query(sql,val)

	return True

@app.route("/litepi",methods = ['POST', 'GET'])
def litepi(boardid=0,boardid_ref=0,comp_selected_list=[],my_designs="All",my_interfaces="All"):

	wwid = session.get('wwid')

	if is_localhost:
		wwid=99999999

	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	# to check for user status - active or inactive
	if session.get('is_inactive_user') or (wwid is None):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	sql="SELECT RoleID FROM HomeTable WHERE WWID=%s "
	val = (wwid,)
	role=execute_query(sql,val)[0][0]

	if (role == 14):
		mgt_access=True
	else:
		mgt_access=False

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	sql = "SELECT * FROM BoardDetails WHERE BoardID = %s AND (DesignLeadWWID = %s OR CADLeadWWID = %s)"	
	val = (boardid,wwid,wwid)
	rs_design_layout_lead = execute_query(sql,val)

	is_design_layout_lead = False
	if rs_design_layout_lead != ():
		is_design_layout_lead = True

	tr_display = "none"
	new_file_name = " - "
	ref_file_name = " - "
	is_file_upload = False
	
	file_response = {}
	file_response["error"] = False
	file_response["error_msg"] = ""

	new_file_upload_id = 0
	ref_file_upload_id = 0
	load_project_rev_no = 0

	ref_design_selection = "old"
	new_ref_design_name = ""

	if request.method == "POST":
		boardid = request.form.get("boardid",type=int)
		boardid_ref = request.form.get("boardid_ref",type=int)
		#comp_selected_list = request.form.getlist("comp_select")
		comp_selected_list = request.form.getlist("comp_select_submit")

		new_design_file = request.form.get("new_design_file",type=str)
		ref_design_file = request.form.get("ref_design_file",type=str)

		ref_design_selection = request.form.get("ref_design_selection",type=str)
		new_ref_design_name = request.form.get("new_ref_design_name",type=str)

		load_project_rev_no = request.form.get("load_project",type=int)

		if boardid_ref in [None,0,""]:
			boardid_ref = 0

		if ref_design_selection == "new":
			boardid_ref = 0

		if load_project_rev_no in [0,'0',None,'None']:
			load_project_rev_no = 0

		# assign dropdown file id to proceed further
		if new_design_file not in [0,'0',None,'None',""]:

			new_file_upload_id = copy.deepcopy(new_design_file)

			tr_display = "tr-row"
			is_file_upload = True

			sql = "SELECT FileName FROM LitePiFileStorage WHERE FileID = %s"
			val = (new_file_upload_id,)
			file_rs = execute_query(sql,val)

			if file_rs != ():
				new_file_name = copy.deepcopy(file_rs[0][0])

		if ref_design_file not in [0,'0',None,'None',""]:
			
			ref_file_upload_id = copy.deepcopy(ref_design_file)

			tr_display = "tr-row"
			is_file_upload = True

			sql = "SELECT FileName FROM LitePiFileStorage WHERE FileID = %s"
			val = (ref_file_upload_id,)
			file_rs = execute_query(sql,val)

			if file_rs != ():
				ref_file_name = copy.deepcopy(file_rs[0][0])

		if (request.files["new_file_upload"]) and (boardid not in [None,""]) and (str(new_file_upload_id) == "0"):

			file = request.files["new_file_upload"]
			file_name = secure_filename(file.filename)

			tr_display = "tr-row"
			is_file_upload = True
			new_file_name = copy.deepcopy(file_name)

			file_response = litepi_file_process(boardid=boardid,file_name=file_name,file_upload=file)
			new_file_upload_id = file_response["file_id"]

		if (request.files["ref_file_upload"]) and ((boardid_ref not in [None,"",0]) or (new_ref_design_name != "")) and (str(ref_file_upload_id) == "0"):

			file = request.files["ref_file_upload"]
			file_name = secure_filename(file.filename)

			tr_display = "tr-row"
			is_file_upload = True
			ref_file_name = copy.deepcopy(file_name)

			file_response = litepi_file_process(boardid=boardid_ref,file_name=file_name,file_upload=file)
			ref_file_upload_id = file_response["file_id"]


	# check for any errors in file processing 
	if file_response["error"]:
		tr_display = "none"
		is_file_upload = False
		print("Error in File Processing: ",file_response["error_msg"])


	# new design
	design_status_list = []
	design_list = []
	my_designs_id = []

	temp_data = []
	temp_data = get_my_designs_litePi()
	my_designs_all_checked = ""
	my_designs_my_checked = "checked"       

	for i in range(0,len(temp_data[1])):
		my_designs_id.append(temp_data[1][i][0])

	if my_designs == "All":
		temp_data = get_all_designs_LitePi()
		my_designs_all_checked = "checked"
		my_designs_my_checked = ""

	#design_status_list = get_status_list_sorted(data_list=temp_data[0])
	design_status_list = get_order_status_list(list=temp_data[0])

	design_list = temp_data[1]

	comp_status_list = []
	comp_list = []
	my_components_id = []

	temp_data = []

	# for design & Layout Lead, Managers - both My interface and All interface should be same and have edit access, as we dont have any mapping for design/layout lead and managers at backend properly
	if is_design_layout_lead:
		temp_data = get_all_interfaces_litepi(boardid=boardid)
	else:
		temp_data = get_my_interfaces_litepi(boardid=boardid)

	my_interfaces_all_checked = ""
	my_interfaces_my_checked = "checked"    

	#for i in range(0,len(temp_data[1])):
	#	my_components_id.append(temp_data[1][i][0])

	if my_interfaces == "All":
		temp_data = []
		temp_data = get_all_interfaces_litepi(boardid=boardid)
		my_interfaces_all_checked = "checked"
		my_interfaces_my_checked = ""

	comp_status_list = get_order_status_list(list=temp_data[0])

	comp_list = temp_data[1]

	# Reference design
	design_status_list_ref = []
	design_list_ref = []
	#my_designs_id_ref = []

	temp_data_ref = []
	temp_data_ref = get_all_designs_feedbacks()

	#for i in range(0,len(temp_data_ref[1])):
	#	my_designs_id_ref.append(temp_data_ref[1][i][0])

	#design_status_list = get_status_list_sorted(data_list=temp_data[0])
	design_status_list_ref = get_order_status_list(list=temp_data_ref[0])

	design_list_ref = temp_data_ref[1]

	vrm_list = []
	vrm_list_ref = []
	net_list = []
	net_list_ref = []

	vrm_list = get_vrm_list(boardid=boardid,file_id=new_file_upload_id)
	vrm_list_ref = get_vrm_list(boardid=boardid_ref,file_id=ref_file_upload_id)

	net_list = get_net_list(boardid=boardid,file_id=new_file_upload_id)
	net_list_ref = get_net_list(boardid=boardid_ref,file_id=ref_file_upload_id)

	gnd_net_list = get_gnd_net_list(boardid=boardid,file_id=new_file_upload_id)
	gnd_net_list_ref = get_gnd_net_list(boardid=boardid_ref,file_id=ref_file_upload_id)

	if ref_design_selection == "new":
		existing_ref_design = ""
		new_ref_design = "checked"
		boardid_ref_div_display = "none"
		new_ref_design_name_div = "block"

	else:
		existing_ref_design = "checked"
		new_ref_design = ""
		new_ref_design_name = ""
		boardid_ref_div_display = "block"
		new_ref_design_name_div = "none"

	sql = "SELECT ConfigJson FROM LitePiProjects WHERE BoardID = %s AND RunNo = %s"
	val = (boardid,load_project_rev_no)
	load_project_rs = execute_query(sql,val)

	print(load_project_rs)

	load_project = {}
	is_load_project = False

	try:
		if load_project_rs != ():
			load_project = json.loads(load_project_rs[0][0])

			if "new_design" in load_project.keys():
				is_load_project = True

				comp_selected_list = load_project["comp_selected_list"]

				print("comp_selected_list inside load project: ",comp_selected_list)

	except Exception as inst:
		print("Error in loading project: ",inst)

	print(type(load_project))

	return render("lite_pi.html",is_prod=is_prod,file_response=file_response,tr_display=tr_display,new_file_upload_id=new_file_upload_id,ref_file_upload_id=ref_file_upload_id,is_file_upload=is_file_upload,new_file_name=new_file_name,ref_file_name=ref_file_name,boardid=boardid,boardid_ref=boardid_ref,design_status_list_ref=design_status_list_ref,design_list_ref=design_list_ref,comp_selected_list=comp_selected_list,is_admin=is_admin,is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner,username=username,user_role_name=user_role_name,region_name=region_name,admin='yes',mgt_access=mgt_access,design_status_list=design_status_list,design_list=design_list,comp_status_list=comp_status_list,comp_list=comp_list,my_designs_all_checked=my_designs_all_checked,my_designs_my_checked=my_designs_my_checked,vrm_list=vrm_list,net_list=net_list,vrm_list_ref=vrm_list_ref,net_list_ref=net_list_ref,gnd_net_list=gnd_net_list,gnd_net_list_ref=gnd_net_list_ref,existing_ref_design=existing_ref_design,new_ref_design=new_ref_design,new_ref_design_name=new_ref_design_name,boardid_ref_div_display=boardid_ref_div_display,new_ref_design_name_div=new_ref_design_name_div,is_load_project=is_load_project,load_project=load_project)

def litepi_file_process(boardid,file_name,file_upload):

	wwid = session.get('wwid')
	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	#file_path = cloud_base_url + "//litepi//uploaded_files//Board_ID_"+str(boardid)+"//"+str(file_name)
	file_path = os.path.join(cloud_base_url,"litepi","uploaded_files","Board_ID_"+str(boardid),str(file_name))
	spd_file_path = ""
	brd_file_path = ""

	result = {}
	result["error"] = False
	result["error_msg"] = ""
	result["file_id"] = 0

	file_type = ""
	is_valid_file_found_in_zip = False

	try:
		#if not os.path.exists(cloud_base_url + "//litepi//uploaded_files//Board_ID_" + str(boardid)):
		#	os.makedirs(cloud_base_url + "//litepi//uploaded_files//Board_ID_" + str(boardid))

		if not os.path.exists(os.path.join(cloud_base_url,"litepi","uploaded_files","Board_ID_"+str(boardid))):
			os.makedirs(os.path.join(cloud_base_url,"litepi","uploaded_files","Board_ID_"+str(boardid)))
	
	except Exception as inst:
		if is_logging:
			logging.exception('')
		print("Error in creating path. Error: ",inst)

		result["error"] = True
		result["error_msg"] = "Error in creating path."
		return result

	if file_name.lower().endswith('.brd'):
		file_type = "brd"
	
	elif file_name.lower().endswith('.spd'):
		file_type = "spd"

	elif file_name.lower().endswith('.zip'):
		file_type = "zip"
		#zip_file_path_temp = cloud_base_url + "//litepi//uploaded_files//zip_temp//Board_ID_"+str(boardid)+"//zip"+str(random.randint(1,1000))
		zip_file_path_temp = os.path.join(cloud_base_url,"litepi","uploaded_files","zip_temp","Board_ID_"+str(boardid),"zip"+str(random.randint(1,1000)))

		try:
			if not os.path.exists(zip_file_path_temp):
				os.makedirs(zip_file_path_temp)
		
		except Exception as inst:
			if is_logging:
				logging.exception('')
			print("Error in creating zip file path. Error: ",inst)

			result["error"] = True
			result["error_msg"] = "Error in creating zip file path."
			return result

		# save zip file
		try:
			file_upload.save(file_path)

		except Exception as inst:
			if is_logging:
				logging.exception('')
			print("File save error: ",inst)

			result["error"] = True
			result["error_msg"] = "Error in Saving File."
			return result

		# extract zip file
		with zipfile.ZipFile(file_path,'r') as zip_file:
			zip_file.extractall(zip_file_path_temp)

		# read extracted zip file and look for spd file first, if not then brd file
		extracted_files = os.listdir(zip_file_path_temp)

		# look for spd file
		for row in extracted_files:
			file_path_temp = copy.deepcopy(os.path.join(zip_file_path_temp,row))
			#file_path = cloud_base_url + "//litepi//uploaded_files//Board_ID_"+str(boardid)+"//"+str(row)
			file_path = os.path.join(cloud_base_url,"litepi","uploaded_files","Board_ID_"+str(boardid),str(row))

			print("file_path: ",file_path)

			if file_path_temp.endswith(".spd"):
				print("spd file found")
				file_type = "spd"
				file_name = file_name.replace(".zip",".spd")
				is_valid_file_found_in_zip = True

				# copy this valid file to main stroage directory
				shutil.copy(file_path_temp,file_path)

				break

		# look for brd file, if no spd file found
		if not is_valid_file_found_in_zip:
			
			for row in extracted_files:
				file_path_temp = copy.deepcopy(os.path.join(zip_file_path_temp,row))
				#file_path = cloud_base_url + "//litepi//uploaded_files//Board_ID_"+str(boardid)+"//"+str(row)
				file_path = os.path.join(cloud_base_url,"litepi","uploaded_files","Board_ID_"+str(boardid),str(row))
				print("file_path : ",file_path)

				if file_path_temp.endswith(".brd"):
					print("brd file found")
					file_type = "brd"
					file_name = file_name.replace(".zip",".brd")
					is_valid_file_found_in_zip = True

					# copy this valid file to main stroage directory
					shutil.copy(file_path_temp,file_path)

					break

		if not is_valid_file_found_in_zip:

			result["error"] = True
			result["error_msg"] = "No valid files found inside the uploaded zip file."
			return result

		for row in extracted_files:
			try:
				os.remove(os.path.join(zip_file_path_temp,row))
				print("Extracted file removed. File: ",os.path.join(zip_file_path_temp,row))
			except Exception as inst:
				if is_logging:
					logging.exception('')
				print("Error removing file. Error: ",inst)

	else:
		result["error"] = True
		result["error_msg"] = "Invalid File Type."
		return result

	# save file only if direct brd or spd file upload, but not from zip
	if not is_valid_file_found_in_zip:

		try:
			file_upload.save(file_path)

		except Exception as inst:
			if is_logging:
				logging.exception('')
			print("File save error: ",inst)

			result["error"] = True
			result["error_msg"] = "Error in Saving File."
			return result

	# check for file type 
	if file_type == "spd":
		spd_file_path = copy.deepcopy(file_path)

	# convert board to spd file format
	elif file_type == "brd":
		brd_file_path = copy.deepcopy(file_path)
		
		remote_dir = remote_linux_brd_to_spd_base_path + "R" + str(random.randint(1,1000))
		#remote_file_path = remote_dir+"//"+str(file_name)
		remote_file_path = os.path.join(remote_dir,str(file_name))
		#print("remote_file_path: ",remote_file_path)
		print("Pushing board file to linux server for SPD convertion...")

		# pushing file to server to linux server for brd to spd convertion
		if not push_file_to_linux_server(boardid=boardid,local_file_path=brd_file_path,remote_file_path=remote_file_path,remote_dir=remote_dir):
			print("Error pushing Board file to linux server for SPD convertion.")

			result["error"] = True
			result["error_msg"] = "Error pushing Board file to Server for SPD convertion."
			return result
		
		brd_to_spd_file_cmd = cadence_license_update_cmd + ';' + cmd_powersi_path + ' -noui ' + remote_file_path

		# execute command for brd to spd convertion on linux server
		if not exec_command_through_ssh(server="linux",cmd=brd_to_spd_file_cmd,need_response=True,close_ssh=True):
			print("Error in board to spd file convertion.")

			result["error"] = True
			result["error_msg"] = "Error in Board to SPD File convertion."
			return result

		time.sleep(1)

		#local_folder = cloud_base_url + "//litepi//uploaded_files//Board_ID_" + str(boardid)
		local_folder = os.path.join(cloud_base_url,"litepi","uploaded_files","Board_ID_"+str(boardid))

		# create directory if not exists on local SMB disk
		try:
			if not os.path.exists(local_folder):
				os.makedirs(local_folder)
		except Exception as inst:
			if is_logging:
				logging.exception('')
			print("Error creating path. Path: ",local_folder,". Error: ",inst)

			result["error"] = True
			result["error_msg"] = "Error in creating a path."
			return result


		# create ssh connection
		try:
			ssh = get_server_machine_connection(server="linux")
			sftp_client = ssh.open_sftp()
		except Exception as inst:
			if is_logging:
				logging.exception('')
			print("Error opening SSH connection for Board to SPD file convertion. Error: ",inst)

			result["error"] = True
			result["error_msg"] = "Error in opening SSH connection."
			return result


		print("Source path: ",remote_dir)
		inbound_files = sftp_client.listdir(remote_dir)

		# copy file from remote to local
		for ele in inbound_files:

			# check for completed execution
			if (".spd" in ele) and (".spdif" not in ele):
				print("Board to SPD file convertion Completed successfully.")
				#remote_spd_file_path = remote_dir+"//"+ele
				#spd_file_path = local_folder+"//"+ele

				remote_spd_file_path = os.path.join(remote_dir,ele)
				spd_file_path = os.path.join(local_folder,ele)

				#print("remote_spd_file_path: ",remote_spd_file_path)
				#print("local_spd_file_path: ",spd_file_path)

				# copy file from remote linux server to local SMB disk
				try:
					sftp_client.get(remote_spd_file_path, spd_file_path)
					print("Spd file copied from remote server to local SMB disk.")
				except Exception as inst:
					if is_logging:
						logging.exception('')
					print("Error copying SPD file from remote server to local SMB disk. Error: ",inst)

					result["error"] = True
					result["error_msg"] = "Error in fetching SPD file from server."
					return result

		
		time.sleep(1)

		# remove file from remote linux server to local SMB disk
		for ele in inbound_files:

			#remote_spd_file_path = remote_dir+"//"+ele
			remote_spd_file_path = os.path.join(remote_dir,ele)

			try:
				# remove file from remote linux server to local SMB disk
				sftp_client.remove(remote_spd_file_path)		
				print("File removed: ",remote_spd_file_path)
			except Exception as inst:
				if is_logging:
					logging.exception('')
				print("Error deleting file. Error: ",inst)

		try:
			# terminate ssh connection
			sftp_client.close()
			close_ssh_connection(ssh=ssh)
		except Exception as inst:
			if is_logging:
				logging.exception('')
			print("Error closing SSH connection for Board to SPD file convertion. Error: ",inst)

	else:
		print("Invalid File Type.")

		result["error"] = True
		result["error_msg"] = "Invalid File Type"
		return result


	# end of brd to spd file convetion section


	file_data = ""

	try:
		with open(spd_file_path) as f:
			file_data = f.readlines()
			f.close()
	except Exception as inst:
		if is_logging:
			logging.exception('')
		print("Error reading SPD file: ",inst)

		result["error"] = True
		result["error_msg"] = "Error in reading SPD File"
		return result

	if len(file_data) < 10:
		print("Invalid SPD file data.")

		result["error"] = True
		result["error_msg"] = "Invalid SPD file data."
		return result

	#upload_file = file_upload.read()

	sql = "INSERT INTO LitePiFileStorage VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s)"
	val = (None,boardid,file_name,file_type,None,brd_file_path,spd_file_path,t,wwid)
	execute_query(sql,val)

	sql = "SELECT LAST_INSERT_ID()"
	result["file_id"] = execute_query_sql(sql)[0][0]

	vrm_list = []
	vrm_list = get_vrm_list_from_spd_file(boardid=boardid,file_id=result["file_id"],data=file_data)

	if (len(vrm_list) == 0):
		result["error"] = True
		result["error_msg"] = "Invalid SPD file data. VRM details are not found."
		return result

	net_list = []
	net_list = get_net_list_from_spd_file(boardid=boardid,file_id=result["file_id"],data=file_data)

	if (len(net_list) == 0):
		result["error"] = True
		result["error_msg"] = "Invalid SPD file data. Net details are not found."
		return result

	stackup_details = []
	stackup_details = get_stackup_details_from_spd_file(boardid=boardid,file_id=result["file_id"],data=file_data)

	if (len(stackup_details) == 0):
		result["error"] = True
		result["error_msg"] = "Invalid SPD file data. Stackup details are not found."
		return result

	result["error"] = False
	result["error_msg"] = ""
	return result


def get_all_interfaces_litepi(boardid=0):

	wwid=session.get('wwid')
	username = session.get('username')

	comp_status_list = []
	comp_list = []
	pi_category_ids = (1,2,14)

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	if sku_plat != ():
		sql = "SELECT DISTINCT S2.ScheduleTypeName FROM BoardReviewDesigner B1, ComponentType C1, ComponentReview C2, ScheduleTableComponent S1,ScheduleStatusType S2,CategoryType C3 WHERE B1.ComponentID = C1.ComponentID AND C1.CategoryID = C3.CategoryID AND B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C2.ComponentID AND C2.IsValid = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID = %s AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S2.ScheduleID = S1.ScheduleStatusID AND C3.CategoryID IN %s ORDER BY FIELD(S1.ScheduleStatusID,2,6,3,7,5,1,4), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
		val = (boardid,"yes",True,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],pi_category_ids)
		result = execute_query(sql,val)

		if result != ():
			for i in range(0,len(result)):
				comp_status_list.append(result[i][0])

		sql = "SELECT DISTINCT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1, ComponentType C1, ComponentReview C2, ScheduleTableComponent S1,ScheduleStatusType S2,CategoryType C3 WHERE B1.ComponentID = C1.ComponentID AND C1.CategoryID = C3.CategoryID AND B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C2.ComponentID AND C2.IsValid = %s AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID = %s AND S1.BoardID = B1.BoardID AND S1.ComponentID = B1.ComponentID AND S2.ScheduleID = S1.ScheduleStatusID AND C3.CategoryID IN %s ORDER BY FIELD(S1.ScheduleStatusID,2,6,3,7,5,1,4), FIELD(C1.CategoryID, 11,2,1,4,3,5,6,7,8,9,10,12,14,15,16,17,18,19,13), C1.ComponentName"
		val = (boardid,"yes",True,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],pi_category_ids)
		complist = execute_query(sql,val)

		if complist != ():
			for i in range(0,len(complist)):
				temp = [complist[i][0],complist[i][1],complist[i][2],complist[i][3]]
				comp_list.append(temp)

	comp_status_list = get_order_status_list(list=comp_status_list)

	return [comp_status_list,comp_list]

def get_my_interfaces_litepi(boardid=0):

	wwid=session.get('wwid')
	username = session.get('username')

	comp_list = []
	comp_status_list = []
	pi_category_ids = (1,2,14)

	like_wwid = '%' + str(wwid) + '%'

	sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	sku_plat = execute_query(sql,val)

	if sku_plat != ():

		sql = "SELECT DISTINCT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1, ComponentType C1, ComponentReview C2, ScheduleTableComponent S1,ScheduleStatusType S2,CategoryType C3 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND C1.CategoryID = C3.CategoryID AND C2.IsValid = %s AND B1.ComponentID = C2.ComponentID AND B1.BoardID = S1.BoardID AND B1.ComponentID = S1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID =%s AND (C2.PrimaryWWID = %s OR C2.SecondaryWWID LIKE %s) AND C3.CategoryID IN %s"

		sql += " UNION "
		sql += "SELECT DISTINCT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1, ComponentType C1, ComponentDesign C2, ScheduleTableComponent S1,ScheduleStatusType S2,CategoryType C3 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND C1.CategoryID = C3.CategoryID AND C2.IsValid = %s AND B1.ComponentID = C2.ComponentID AND B1.BoardID = S1.BoardID AND B1.ComponentID = S1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID =%s AND (C2.PrimaryWWID = %s OR C2.SecondaryWWID LIKE %s) AND C3.CategoryID IN %s"

		sql += " UNION "

		sql += "SELECT DISTINCT B1.ComponentID,C1.ComponentName,S2.ScheduleTypeName,S1.ScheduleStatusID FROM BoardReviewDesigner B1, ComponentType C1, CategoryLeadTable C2, ScheduleTableComponent S1,ScheduleStatusType S2,CategoryType C3 WHERE B1.BoardID = %s AND B1.IsPdgDesignSubmitted = %s AND B1.ComponentID = C1.ComponentID AND C1.CategoryID = C2.CategoryID AND C1.CategoryID = C3.CategoryID AND B1.BoardID = S1.BoardID AND B1.ComponentID = S1.ComponentID AND S1.ScheduleStatusID = S2.ScheduleID AND C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID =%s AND C2.DesignTypeID =%s AND (C2.CategoryLeadWWID = %s OR C2.CategoryLeadWWID1 LIKE %s) AND C3.CategoryID IN %s"
		val = (boardid,"yes",True,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],wwid,like_wwid,pi_category_ids,boardid,"yes",True,sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],wwid,like_wwid,pi_category_ids,boardid,"yes",sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],wwid,like_wwid,pi_category_ids)

		complist = execute_query(sql,val)

		if complist != ():
			for i in range(0,len(complist)):
				temp = [complist[i][0],complist[i][1],complist[i][2],complist[i][3]]
				comp_list.append(temp)
				comp_status_list.append(complist[i][2])

		comp_status_list = list(set(comp_status_list))

	comp_status_list = get_order_status_list(list=comp_status_list)

	return [comp_status_list,comp_list]

@app.route("/get_feedbacks_comp_litepi",methods = ['POST', 'GET'])
def get_feedbacks_comp_litepi():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))

	boardid = copy.deepcopy(data['boardid'])
	my_interfaces = copy.deepcopy(data['my_interfaces'])

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)[0][0]

	sql = "SELECT RoleID FROM HomeTable WHERE WWID = %s"
	val = (wwid,)
	role = execute_query(sql,val)[0][0]

	is_elec_owner = 'no'
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = 'yes'

	sql = "SELECT * FROM BoardDetails WHERE BoardID = %s AND (DesignLeadWWID = %s OR CADLeadWWID = %s)"	
	val = (boardid,wwid,wwid)		
	rs_design_layout_lead = execute_query(sql,val)

	is_design_layout_lead = False
	if rs_design_layout_lead != ():
		is_design_layout_lead = True

	comp_status_list = []
	comp_list = []
	feedbacks = []

	if my_interfaces == "All":
		temp_data = []
		#temp_data = get_all_interfaces_feedbacks(boardid=boardid)
		temp_data = get_all_interfaces_litepi(boardid=boardid)

	else:
		temp_data = []
		if is_design_layout_lead:
			#temp_data = get_all_interfaces_feedbacks(boardid=boardid)
			temp_data = get_all_interfaces_litepi(boardid=boardid)
		else:
			#temp_data = get_my_interfaces_feedbacks(boardid=boardid)		
			temp_data = get_my_interfaces_litepi(boardid=boardid)		

	comp_list = temp_data[1]
	#comp_status_list = get_status_list_sorted(data_list=temp_data[0])
	comp_status_list = get_order_status_list(list=temp_data[0])

	# get filenames
	#sql = "SELECT FileID,FileName FROM LitePiFileStorage WHERE BoardID = %s"
	sql = "SELECT A.FileID, LOWER(A.FileName) FROM LitePiFileStorage A WHERE A.BoardID = %s AND (A.FileID,A.FileName) IN (SELECT MAX(B.FileID),B.FileName FROM LitePiFileStorage B WHERE A.BoardID = B.BoardID GROUP BY B.FileName) ORDER BY A.FileName"
	val = (boardid,)
	file_rs = execute_query(sql,val)

	file_list = []

	for i in range(len(file_rs)):
		temp = []
		temp.append(file_rs[i][0])
		temp.append(file_rs[i][1])
		
		file_list.append(temp)

	sql = "SELECT ScheduleStatusID FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)		
	rs_design_status = execute_query(sql,val)

	# disable new litepi run for signed-off design
	is_invalid_design = False
	if rs_design_status != ():
		if rs_design_status[0][0] in [1,'1']:
			is_invalid_design = True

	# get saved project details
	sql = "SELECT RunNo FROM LitePiProjects WHERE BoardID = %s ORDER BY RunNo DESC"
	val = (boardid,)
	load_project_rs = execute_query(sql,val)

	load_project_list = []
	
	for row in load_project_rs:
		temp=[]
		temp.append(row[0])
		temp.append('Rev'+str(row[0]))
		
		load_project_list.append(temp)

	final_result = [comp_status_list,comp_list,file_list,is_invalid_design,load_project_list]

	return jsonify(final_result)

@app.route("/get_litepi_project_details",methods = ['POST', 'GET'])
def get_litepi_project_details():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))

	boardid = copy.deepcopy(data['boardid'])
	run_no = copy.deepcopy(data['run_no'])

	# get filenames
	#sql = "SELECT FileID,FileName FROM LitePiFileStorage WHERE BoardID = %s"
	sql = "SELECT A.FileID, LOWER(A.FileName) FROM LitePiFileStorage A WHERE A.BoardID = %s AND (A.FileID,A.FileName) IN (SELECT MAX(B.FileID),B.FileName FROM LitePiFileStorage B WHERE A.BoardID = B.BoardID GROUP BY B.FileName) ORDER BY A.FileName"
	val = (boardid,)
	file_rs = execute_query(sql,val)

	file_list = []
	ref_file_list = []

	for i in range(len(file_rs)):
		temp = []
		temp.append(file_rs[i][0])
		temp.append(file_rs[i][1])
		
		file_list.append(temp)

	sql = "SELECT ConfigJson FROM LitePiProjects WHERE BoardID = %s AND RunNo = %s"
	val = (boardid,run_no)
	config_json_rs = execute_query(sql,val)

	selected_file_id = 0
	ref_design_id = 0

	if config_json_rs != ():
		try:
			config_json = json.loads(config_json_rs[0][0])

			selected_file_id = config_json["new_design"]["file_id"]
			ref_design_id = config_json["ref_design"]["design_id"]
			ref_design_name = config_json["ref_design"]["design_name"]
			ref_file_id = config_json["ref_design"]["file_id"]

		except Exception as inst:
			print(inst)


	if ref_design_id != 0:
		# get filename for reference design
		#sql = "SELECT FileID,FileName FROM LitePiFileStorage WHERE BoardID = %s"
		sql = "SELECT A.FileID, LOWER(A.FileName) FROM LitePiFileStorage A WHERE A.BoardID = %s AND (A.FileID,A.FileName) IN (SELECT MAX(B.FileID),B.FileName FROM LitePiFileStorage B WHERE A.BoardID = B.BoardID GROUP BY B.FileName) ORDER BY A.FileName"
		val = (ref_design_id,)
		file_rs = execute_query(sql,val)

		for i in range(len(file_rs)):
			temp = []
			temp.append(file_rs[i][0])
			temp.append(file_rs[i][1])
			
			ref_file_list.append(temp)

	final_result = [file_list,selected_file_id,ref_design_id,ref_design_name,ref_file_id,ref_file_list]

	return jsonify(final_result)

@app.route("/get_design_status_litepi",methods = ['POST', 'GET'])
def get_design_status_litepi():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))

	boardid = copy.deepcopy(data['boardid'])

	sql = "SELECT ScheduleStatusID FROM ScheduleTable WHERE BoardID = %s"
	val = (boardid,)		
	rs_design_status = execute_query(sql,val)

	# disable new litepi run for signed-off design
	is_invalid_design = False
	if rs_design_status != ():
		if rs_design_status[0][0] in [1,'1']:
			is_invalid_design = True

	final_result = [is_invalid_design]

	return jsonify(final_result)

@app.route("/get_litepi_file_list",methods = ['POST', 'GET'])
def get_litepi_file_list():

	wwid=session.get('wwid')
	username = session.get('username')

	data = json.loads(request.form.get("data"))

	boardid = copy.deepcopy(data['boardid'])

	# get filenames
	#sql = "SELECT FileID,FileName FROM LitePiFileStorage WHERE BoardID = %s"
	sql = "SELECT A.FileID, LOWER(A.FileName) FROM LitePiFileStorage A WHERE A.BoardID = %s AND (A.FileID,A.FileName) IN (SELECT MAX(B.FileID),B.FileName FROM LitePiFileStorage B WHERE A.BoardID = B.BoardID GROUP BY B.FileName) ORDER BY A.FileName"
	val = (boardid,)
	file_rs = execute_query(sql,val)

	file_list = []

	for i in range(len(file_rs)):
		temp = []
		temp.append(file_rs[i][0])
		temp.append(file_rs[i][1])
		
		file_list.append(temp)

	return jsonify(file_list)

@app.route("/get_vrm_list",methods = ['POST', 'GET'])
def get_vrm_list(boardid=0,file_id=0):

	wwid=session.get('wwid')
	username = session.get('username')

	#boardid = request.form.get("boardid")

	sql = "SELECT VrmName FROM LitePiVrmList WHERE BoardID = %s AND FileID = %s ORDER BY SequenceNo"
	val = (boardid,file_id)
	result_rs = execute_query(sql,val)

	return result_rs

@app.route("/get_net_list",methods = ['POST', 'GET'])
def get_net_list(boardid=0,file_id=0):

	wwid=session.get('wwid')
	username = session.get('username')

	#boardid = request.form.get("boardid")

	sql = "SELECT NetName FROM LitePiNetList WHERE BoardID = %s AND FileID = %s ORDER BY SequenceNo"
	val = (boardid,file_id)
	result_rs = execute_query(sql,val)

	return result_rs

@app.route("/get_gnd_net_list",methods = ['POST', 'GET'])
def get_gnd_net_list(boardid=0,file_id=0):

	wwid=session.get('wwid')
	username = session.get('username')

	#boardid = request.form.get("boardid")

	sql = "SELECT NetName FROM LitePiNetList WHERE BoardID = %s AND FileID = %s AND (NetName LIKE %s OR NetName LIKE %s) ORDER BY SequenceNo"
	val = (boardid,file_id,"%VSS%","%GND%")
	result_rs = execute_query(sql,val)

	return result_rs

@app.route("/get_interface_level_litepi_run_details",methods = ['POST', 'GET'])
def get_interface_level_litepi_run_details():

	wwid=session.get('wwid')
	username = session.get('username')
	
	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]
	run_no = data["run_no"]

	result = {}

	result["boardid"] = copy.deepcopy(boardid)
	result["run_no"] = copy.deepcopy(run_no)
	result["board_name"] = ""
	result["litepi_run_details"] = []
	result["rev_name"] = "Rev"+str(run_no)
	result["enable_download_all"] = False

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	result_rs = execute_query(sql,val)

	if result_rs != ():
		result["board_name"] = result_rs[0][0]

	sql = "SELECT * FROM LitePiReportsFileStorage a WHERE a.BoardID = %s AND a.RunNo = %s AND a.ComponentID = %s"
	val = (boardid,run_no,0)
	rs = execute_query(sql,val)

	if rs != ():
		result["enable_download_all"] = True


	sql = "SELECT a.BoardID,a.RunNo,a.ComponentID,c.ComponentName,a.IsRunning,a.Status FROM LitePiRunInterfaceLevelDetails a,ComponentType c WHERE a.BoardID = %s AND a.RunNo = %s AND a.ComponentID = c.ComponentID ORDER BY a.Status,c.LitePiComponentName"
	val = (boardid,run_no)
	result_rs = execute_query(sql,val)

	for row in result_rs:
		temp = []
		temp.append(row[0])
		temp.append(row[1])
		temp.append(row[2])
		temp.append(row[3])

		# for download option - 4
		if (row[5] in ["Completed","Error"]):
			temp.append(True)
		else:
			temp.append(False)

		# for status - 5
		if row[5] == "Completed":
			temp.append('<font color="green"><b>Completed</b></font>')

		elif row[5] == "Error":
			temp.append('<font color="red"><b>Error</b></font>')
			
		elif row[5] == "Pending":
			temp.append('<font color="#CC338B"><b>Pending</b></font>')

		else:
			temp.append('<font color="#D9B611"><b>Running</b></font>')

		result["litepi_run_details"].append(temp)

	return jsonify(result)

@app.route("/get_stackup_details",methods = ['POST', 'GET'])
def get_stackup_details():

	wwid=session.get('wwid')
	username = session.get('username')
	
	data = json.loads(request.form.get("data"))
	
	boardid = data["boardid"]
	file_id = data["file_id"]
	print("get_stackup_details - boardid: ",boardid)
	print("get_stackup_details - file_id: ",file_id)

	result = {}

	result["boardid"] = copy.deepcopy(boardid)
	result["file_id"] = copy.deepcopy(file_id)
	result["board_name"] = ""
	result["stackup_details"] = []

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	result_rs = execute_query(sql,val)

	if result_rs != ():
		result["board_name"] = result_rs[0][0]

	sql = "SELECT BoardID,StackupName,LayerType,Thickness,Conductivity,Permittivity,LossTangent,UpdatedOn,UpdatedBy FROM LitePiStackupDetails WHERE BoardID = %s AND FileID = %s ORDER BY SequenceNo"
	val = (boardid,file_id)
	result_rs = execute_query(sql,val)

	for row in result_rs:
		temp = []
		temp.append(row[0])
		temp.append(row[1])
		temp.append(row[2])
		temp.append(row[3])
		temp.append(row[4])
		temp.append(row[5])
		temp.append(row[6])
		temp.append(row[7])
		temp.append(row[8])

		result["stackup_details"].append(temp)

	return jsonify(result)

@app.route("/get_stackup_details_json",methods = ['POST', 'GET'])
def get_stackup_details_json(boardid=0,file_id=0):

	wwid=session.get('wwid')
	username = session.get('username')

	result = []

	sql = "SELECT BoardID,StackupName,LayerType,Thickness,Conductivity,Permittivity,LossTangent,UpdatedOn,UpdatedBy FROM LitePiStackupDetails WHERE BoardID = %s AND FileID = %s ORDER BY SequenceNo"
	val = (boardid,file_id)
	result_rs = execute_query(sql,val)

	for row in result_rs:
		temp = []
		temp.append(row[0])
		temp.append(row[1])
		temp.append(row[2])
		temp.append(row[3])
		temp.append(row[4])
		temp.append(row[5])
		temp.append(row[6])
		temp.append(row[7])
		temp.append(row[8])

		result.append(temp)

	return result

def get_vrm_list_from_spd_file(boardid=0,file_id=0,data=[]):
	'''
	Preparing vrm list from spd file
	'''

	vrm_list = []

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
	wwid = 99999999

	connect_string = ".Connect "
	i=0

	for line in data:
		
		if connect_string in line:

			str_list = line.split(" ")

			if len(str_list) > 0:

				vrm_name = str_list[1].strip()

				# ignoring all capacitor, resistor and inductor
				#if not (vrm_name.startswith("C") or vrm_name.startswith("R") or vrm_name.startswith("L")):
				if not vrm_name.startswith("TP"):

					i+=1

					temp_vrm_list = []
					temp_vrm_list.append(boardid)
					temp_vrm_list.append(file_id)
					temp_vrm_list.append(i)
					temp_vrm_list.append(vrm_name)
					#temp_vrm_list.append(t)
					#temp_vrm_list.append(wwid)

					vrm_list.append(tuple(temp_vrm_list))

	print("VRM list count: ", len(vrm_list))

	sql = "DELETE FROM LitePiVrmList WHERE BoardID = %s AND FileID = %s"
	val = (boardid,file_id)
	execute_query(sql,val)

	if len(vrm_list) > 0:
		sql = """INSERT INTO LitePiVrmList(BoardID,FileID,SequenceNo,VrmName) VALUES (%s,%s,%s,%s)"""
		execute_query_many(sql,vrm_list)

	return vrm_list

def get_net_list_from_spd_file(boardid=0,file_id=0,data=[]):
	'''
	Preparing net list from spd file
	'''

	net_list = []

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
	wwid = 99999999

	net_list_string = ".NetList"
	end_net_list_string = ".EndNetList"


	for i in range(len(data)):
		
		if net_list_string in data[i]:

			x=0

			for j in range(i+2,len(data)):

				if end_net_list_string in data[j]:
					break
				else:
					str_list = data[j].strip().split(" ")
					#print(str_list)
					if len(str_list[0]) > 0:
						x+=1

						temp_net_list = []
						temp_net_list.append(boardid)
						temp_net_list.append(file_id)
						temp_net_list.append(x)

						sub_str = str_list[0].split("::")
						#print(sub_str[0])
						temp_net_list.append(sub_str[0].strip())

						net_list.append(tuple(temp_net_list))

		elif end_net_list_string in data[i]:
			break

	print("Net list count: ", len(net_list))

	sql = "DELETE FROM LitePiNetList WHERE BoardID = %s AND FileID = %s"
	val = (boardid,file_id)
	execute_query(sql,val)

	if len(net_list) > 0:
		sql = """INSERT INTO LitePiNetList(BoardID,FileID,SequenceNo,NetName) VALUES (%s,%s,%s,%s)"""
		execute_query_many(sql,net_list)

	return net_list

def get_stackup_details_from_spd_file(boardid=0,file_id=0,data=""):
	'''
	Preparing stackup details from spd file
	'''

	stackup_details = []

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
	wwid = 99999999

	thickness = "Thickness"
	conductivity = "Conductivity"
	permittivity = "Permittivity"
	loss_tangent = "LossTangent"

	stackup_list_new_db = "stackupListNewDB"

	di_electric_model = ".DielectricModel"
	metal_model = ".MetalModel"

	temp_stackup_details = []
	default_material_value = {}

	equal_str = " = "
	#start_str_list = ["Medium$","Signal$TOP"]
	#end_str_list = ["*","Node","ConformalLayer"]
	start_str_list = ["* Layer description"]
	end_str_list = ["* ConformalLayer","* Node"]
	ignore_str_list = [".TrapezoidalTraceAngle","PatchSignal$","PatchSignal$"]
	valid_stackup_names = ["Signal","Medium"]

	ignore_line = 0

	# fetching default values for material
	for i in range(len(data)):
		if di_electric_model in data[i]:

			temp_data = data[i].split(" ")
			material_name = temp_data[1][:-1]

			temp_data_value_line = data[i+2].split(" ")

			default_material_value[material_name] = {}
			#default_material_value[material_name][conductivity] = 0
			default_material_value[material_name][permittivity] = float(temp_data_value_line[1])
			default_material_value[material_name][loss_tangent] = float(temp_data_value_line[2][:-1])

			# round-off to integer if not float value present
			if default_material_value[material_name][permittivity] == int(default_material_value[material_name][permittivity]):
				default_material_value[material_name][permittivity] = int(default_material_value[material_name][permittivity])

			if default_material_value[material_name][loss_tangent] == int(default_material_value[material_name][loss_tangent]):
				default_material_value[material_name][loss_tangent] = int(default_material_value[material_name][loss_tangent])

		if metal_model in data[i]:

			temp_data = data[i].split(" ")
			material_name = temp_data[1][:-1]

			temp_data_value_line = data[i+2].split(" ")

			default_material_value[material_name] = {}
			default_material_value[material_name][conductivity] = float(temp_data_value_line[1][:-1])
			#default_material_value[material_name][permittivity] = 1
			#default_material_value[material_name][loss_tangent] = 0

			# round-off to integer if not float value present
			if default_material_value[material_name][conductivity] == int(default_material_value[material_name][conductivity]):
				default_material_value[material_name][conductivity] = int(default_material_value[material_name][conductivity])

	#print("default_material_value: ")
	#print(json.dumps(default_material_value,sort_keys=True,indent=4))

	for i in range(len(data)):

		if temp_stackup_details != []:
			break

		#if (equal_str in data[i]) and (bool([ele for ele in start_str_list if(ele in data[i])])):
		if bool([ele for ele in start_str_list if(ele in data[i])]):

			for j in range(i+1,len(data)):

				if j != ignore_line:

					#if (equal_str in data[j]) and (bool([ele for ele in end_str_list if(ele in data[j])])):
					if bool([ele for ele in end_str_list if(ele in data[j])]):
						break

					temp_stackup_line = ""

					temp_stackup_line = copy.deepcopy(data[j][:-1])

					if "+" == data[j+1][0]:
						temp_stackup_line = temp_stackup_line[:-1]+ " " + copy.deepcopy(data[j+1][1::].lstrip(" "))
						ignore_line = j+1

					if not bool([ele for ele in ignore_str_list if(ele in temp_stackup_line)]):
						#print(temp_stackup_line)
						temp_stackup_details.append(temp_stackup_line.replace(" = "," "))

	#print("temp_stackup_details: ")
	#print(json.dumps(temp_stackup_details,sort_keys=True,indent=4))
	'''
	output format
	[board id,sequence number,stackup name,Layer Type,Thickness (mm),Conductivity,Permittivity,Loss Tangent,timestamp,user wwid]
	'''

	i = 0

	for row in temp_stackup_details:

		i+=1

		# intialization
		temp_list = [boardid,file_id,i,"","",0,0,1,0,t,wwid]

		#list_data = row[:-1].split(" ")
		list_data = row.split(" ")

		#print(list_data)
		#print("           ")

		temp_list[3] = list_data[0]

		temp_list[4] = list_data[0].split("$")[0]

		for index in range(len(list_data)):

			if list_data[index] == thickness:
				thickness_unit = list_data[index+1][-1]

				if thickness_unit == "m":
					temp_list[5] = float(list_data[index+1][0:-1]) * 1000
				else:
					temp_list[5] = float(list_data[index+1][0:-1])

				# round-off to integer value if there are no decimal value
				if temp_list[5] == int(temp_list[5]):
					temp_list[5] = int(temp_list[5])

			if list_data[index] == conductivity:
				temp_list[6] = float(list_data[index+1])

				# round-off to integer value if there are no decimal value
				if temp_list[6] == int(temp_list[6]):
					temp_list[6] = int(temp_list[6])

			if list_data[index] == permittivity:
				temp_list[7] = float(list_data[index+1])

				# round-off to integer value if there are no decimal value
				if temp_list[7] == int(temp_list[7]):
					temp_list[7] = int(temp_list[7])

			if list_data[index] == loss_tangent:
				temp_list[8] = float(list_data[index+1])

				# round-off to integer value if there are no decimal value
				if temp_list[8] == int(temp_list[8]):
					temp_list[8] = int(temp_list[8])

			#print(list_data[index])
			# to get values based on Material
			if list_data[index] in default_material_value:

				try:
					if default_material_value[list_data[index]][conductivity]:
						temp_list[6] = copy.deepcopy(default_material_value[list_data[index]][conductivity])
				except:
					pass

				try:
					if default_material_value[list_data[index]][permittivity]:
						temp_list[7] = copy.deepcopy(default_material_value[list_data[index]][permittivity])
				except:
					pass

				try:
					if default_material_value[list_data[index]][loss_tangent]:
						temp_list[8] = copy.deepcopy(default_material_value[list_data[index]][loss_tangent])
				except:
					pass

		# to filter invalid stackup names
		if temp_list[4] in valid_stackup_names:
			stackup_details.append(tuple(temp_list))

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')
	wwid = 99999999

	print("Stackup details count: ",len(stackup_details))
	if stackup_details != []:
		sql = "DELETE FROM LitePiStackupDetails WHERE BoardID = %s AND FileID = %s"
		val = (boardid,file_id)
		execute_query(sql,val)

		sql = """INSERT INTO LitePiStackupDetails(BoardID,FileID,SequenceNo,StackupName,LayerType,Thickness,Conductivity,Permittivity,LossTangent,UpdatedOn,UpdatedBy) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"""
		execute_query_many(sql,stackup_details)

	return stackup_details

@app.route("/get_designs_ajax_litePi",methods = ['POST', 'GET'])
def get_designs_ajax_litePi():

	wwid=session.get('wwid')
	username = session.get('username')

	data = request.form.get("data")

	result = {}

	design_status_list = []
	design_list = []

	if data == "All":
		temp_data = []
		temp_data = get_all_designs_LitePi()

	else:
		temp_data = []
		temp_data = get_my_designs_litePi()

	#result["design_status_list"] = json.dumps(get_status_list_sorted(data_list=temp_data[0]))
	result["design_status_list"] = json.dumps(get_order_status_list(list=temp_data[0]))
	result["design_list"] = json.dumps(temp_data[1])

	return jsonify(result)

def get_all_designs_LitePi():

	wwid=session.get('wwid')
	username = session.get('username')

	design_list = []
	design_status_list = []

	#sql = "SELECT DISTINCT c.ScheduleTypeName FROM BoardDetails a, ScheduleTable b,ScheduleStatusType c WHERE a.BoardID=b.BoardID AND b.ScheduleStatusID=c.ScheduleID AND b.ScheduleStatusID NOT IN (1,4,5,7) ORDER BY FIELD(b.ScheduleStatusID,2,6,3,7,5,1,4),a.BoardID DESC"
	sql = "SELECT DISTINCT c.ScheduleTypeName FROM BoardDetails a, ScheduleTable b,ScheduleStatusType c WHERE a.BoardID=b.BoardID AND b.ScheduleStatusID=c.ScheduleID AND b.ScheduleStatusID NOT IN (4,5,7) ORDER BY FIELD(b.ScheduleStatusID,2,6,3,7,5,1,4),a.BoardID DESC"
	result = execute_query_sql(sql)

	if result != ():
		for i in range(0,len(result)):
			design_status_list.append(result[i][0])


	#sql = "SELECT a.BoardID,a.BoardName,c.ScheduleTypeName,b.ScheduleStatusID FROM BoardDetails a, ScheduleTable b,ScheduleStatusType c WHERE a.BoardID=b.BoardID AND b.ScheduleStatusID=c.ScheduleID AND b.ScheduleStatusID NOT IN (1,4,5,7) ORDER BY FIELD(b.ScheduleStatusID,2,6,3,7,5,1,4),a.BoardID DESC"
	sql = "SELECT a.BoardID,a.BoardName,c.ScheduleTypeName,b.ScheduleStatusID FROM BoardDetails a, ScheduleTable b,ScheduleStatusType c WHERE a.BoardID=b.BoardID AND b.ScheduleStatusID=c.ScheduleID AND b.ScheduleStatusID NOT IN (4,5,7) ORDER BY FIELD(b.ScheduleStatusID,2,6,3,7,5,1,4),a.BoardID DESC"
	result = execute_query_sql(sql)

	if result != ():
		for i in range(0,len(result)):
			temp = [result[i][0],result[i][1],result[i][2]]
			design_list.append(temp)

	design_status_list = get_order_status_list(list=design_status_list)

	return [design_status_list,design_list]

def get_my_designs_litePi():

	wwid=session.get('wwid')

	if is_localhost:
		wwid='99999999'

	username = session.get('username')

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)

	is_admin = False
	if has_admin_access != ():
		if has_admin_access == "yes":
			is_admin = True

	design_list = []
	design_status_list = []

	#sql = "SELECT B.BoardID FROM BoardDetails B,ScheduleStatusType S1, ScheduleTable S2 WHERE S2.BoardID = B.BoardID AND S1.ScheduleID = S2.ScheduleStatusID AND S2.ScheduleStatusID NOT IN (1,4,5,7) ORDER BY FIELD(S2.ScheduleStatusID,2,6,3,7,5,1,4),B.BoardID DESC"
	sql = "SELECT B.BoardID FROM BoardDetails B,ScheduleStatusType S1, ScheduleTable S2 WHERE S2.BoardID = B.BoardID AND S1.ScheduleID = S2.ScheduleStatusID AND S2.ScheduleStatusID NOT IN (4,5,7) ORDER BY FIELD(S2.ScheduleStatusID,2,6,3,7,5,1,4),B.BoardID DESC"
	bnames = execute_query_sql(sql)

	for j in bnames:

		sql = "SELECT SKUID,PlatformID,MemTypeID,DesignTypeID FROM BoardDetails WHERE BoardID = %s"
		val = (j[0],)
		sku_plat = execute_query(sql,val)

		present = False

		if is_admin:
			present = True

		if(present == False):
			sql = "SELECT * FROM BoardDetails WHERE BoardID = %s AND ((DesignLeadWWID = %s) OR (CADLeadWWID = %s) OR (PIFLeadWWID = %s))"
			val = (j[0],wwid,wwid,wwid)
			result = execute_query(sql,val)

			present = False
			if result != ():
				present = True	

		if(present == False):
			sql = "SELECT * FROM BoardDetails a LEFT JOIN HomeTable b ON a.DesignManagerWWID=b.UserName WHERE BoardID = %s AND b.WWID = %s"
			val = (j[0],wwid)
			result = execute_query(sql,val)

			if result != ():
				present = True	

		# for component review
		if(present == False):

			sql = "SELECT C3.CategoryLeadWWID,C2.PrimaryWWID,C2.SecondaryWWID,C3.CategoryLeadWWID1 FROM  ComponentReview C2,CategoryLeadTable C3,ComponentType C1 WHERE C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C1.ComponentID = C2.ComponentID AND C1.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID=%s"
			val = (sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
			primary_des = execute_query(sql,val)
			for i in primary_des:
				if(str(i[0]) == wwid or str(i[1]) == wwid or wwid in str(i[2]) or wwid in str(i[3])):
					present = True
					break

		# for component design
		if(present == False):

			sql = "SELECT C3.CategoryLeadWWID,C2.PrimaryWWID,C2.SecondaryWWID,C3.CategoryLeadWWID1 FROM  ComponentDesign C2,CategoryLeadTable C3,ComponentType C1 WHERE C2.SKUID = %s AND C2.PlatformID = %s AND C2.MemTypeID = %s AND C2.DesignTypeID = %s AND C1.ComponentID = C2.ComponentID AND C1.CategoryID = C3.CategoryID AND C3.SKUID =%s AND C3.PlatformID = %s AND C3.MemTypeID = %s AND C3.DesignTypeID=%s"
			val = (sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3],sku_plat[0][0],sku_plat[0][1],sku_plat[0][2],sku_plat[0][3])
			primary_des = execute_query(sql,val)
			for i in primary_des:
				if(str(i[0]) == wwid or str(i[1]) == wwid or wwid in str(i[2]) or wwid in str(i[3])):
					present = True
					break

		if(present == True):
			sql = "SELECT B.BoardID,B.BoardName,S1.ScheduleTypeName,S1.ScheduleID FROM BoardDetails B, HomeTable H, DesignCalendar D2, ScheduleStatusType S1, ScheduleTable S2 WHERE B.DesignLeadWWID = H.WWID AND B.BoardID = D2.BoardID AND S2.BoardID = B.BoardID AND S1.ScheduleID = S2.ScheduleStatusID AND B.BoardID = %s ORDER BY S1.ScheduleTypeName  "
			val = (j[0],)
			blist = execute_query(sql,val)

			if blist != ():
				for i in range(0,len(blist)):
					design_list.append([blist[i][0],blist[i][1],blist[i][2]])

					if blist[i][2] not in design_status_list:
						if blist[i][3] not in [3,'3']:
							design_status_list.append(blist[i][2])

	design_status_list = get_order_status_list(list=design_status_list)

	return [design_status_list,design_list]

def get_litepi_project_json():

	result = {}
	result["new_design"] = {}
	result["ref_design"] = {}
	result["advanceSetting"] = {}

	result["new_design"]["design_id"] = 0
	result["new_design"]["design_name"] = ""
	result["new_design"]["file_id"] = 0
	result["new_design"]["gnd_net"] = ""
	result["new_design"]["soc_ref_des"] = ""

	result["ref_design"]["is_new_design"] = False
	result["ref_design"]["design_id"] = 0
	result["ref_design"]["design_name"] = ""
	result["ref_design"]["file_id"] = 0
	result["ref_design"]["gnd_net"] = ""
	result["ref_design"]["soc_ref_des"] = ""

	result["comp_selected_list"] = []
	result["comp_configs"] = []

	# advanceSetting
	result["advanceSetting"]["capPrefix"] = "C"
	result["advanceSetting"]["CompDepth"] = "4"
	result["advanceSetting"]["CpuUsage"] = "2"
	result["advanceSetting"]["IgnoreNetKeyWordList"] = "VAL"
	result["advanceSetting"]["InductorPrefix"] = "L"
	result["advanceSetting"]["isGenerate3DPlots"] = "false"
	result["advanceSetting"]["isXY"] = "true"
	result["advanceSetting"]["max"] = "5"
	result["advanceSetting"]["mean"] = "5"
	result["advanceSetting"]["min"] = "-5"
	result["advanceSetting"]["multiplier"] = "10"
	result["advanceSetting"]["passing"] = "0"
	result["advanceSetting"]["resistorPrefix"] = "R"
	result["advanceSetting"]["stdDev"] = "5"

	return result

@app.route("/litepi_submit",methods = ['POST', 'GET'])
def litepi_submit():

	wwid = session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	sql = "SELECT AdminAccess FROM RequestAccess WHERE wwid = %s AND StateID = 2"
	val = (wwid,)
	has_admin_access = execute_query(sql,val)

	sql="SELECT RoleID FROM HomeTable WHERE WWID=%s "
	val = (wwid,)
	role=execute_query(sql,val)[0][0]

	is_elec_owner = False
	if(role == 4 or role == 1 or role == 2 or role == 10 or role == 12 or role == 14):
		is_elec_owner = True

	is_design_owner = False
	if(role == 3 or role == 5 or role == 6):
		is_design_owner = True

	is_layout_owner = False
	if(role == 7 or role == 8 or role == 9):
		is_layout_owner = True

	run_no = 1
	rev = "Rev"+str(run_no)
	ref_design_selection = ""
	new_ref_design_name = ""

	if request.method == "POST":
		boardid = request.form.get("boardid",type=int)
		boardid_ref = request.form.get("boardid_ref",type=int)
		#comp_selected_list = request.form.getlist("comp_select")
		comp_selected_list = request.form.getlist("comp_select_submit")

		new_file_upload_id = int(request.form.get("new_file_upload_id",type=str))
		ref_file_upload_id = int(request.form.get("ref_file_upload_id",type=str))

		ref_design_selection = request.form.get("ref_design_selection",type=str)
		new_ref_design_name = request.form.get("new_ref_design_name",type=str)

	# check for valid Design ID
	if boardid in [0,None]:
		return "Error Occured."

	new_board_name = ""
	ref_board_name = ""

	sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
	val = (boardid,)
	rs_board_name = execute_query(sql,val)
	
	if rs_board_name != ():
		new_board_name = copy.deepcopy(rs_board_name[0][0])

	# check for new ref design name selected
	if ref_design_selection == "new":
		ref_board_name = copy.deepcopy(new_ref_design_name)
		boardid_ref = 0
	
	else:
		sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
		val = (boardid_ref,)
		rs_board_name = execute_query(sql,val)
		
		if rs_board_name != ():
			ref_board_name = copy.deepcopy(rs_board_name[0][0])

	# Calculate revision number for this run
	sql = "SELECT IFNULL(MAX(RunNo),0) FROM LitePiRunDetails WHERE BoardID = %s"
	val = (boardid,)
	rs = execute_query(sql,val)

	if rs != ():
		run_no += rs[0][0]
		rev = "Rev"+str(run_no)

	result = {}
	save_litepi_project = get_litepi_project_json()
	save_litepi_project["comp_configs"] = {}
	save_litepi_project["is_quick_ir_selected"] = False

	stackup_list_new_db = "stackupListNewDB"
	stackup_list_ref_db = "stackupListReferenceDB"

	all_simulation_details = []

	sr_no=1

	print("comp_selected_list: ",comp_selected_list)
	# simulation list
	for comp_id in comp_selected_list:

		print("comp_id: ",comp_id)
		
		if request.form.get("icc_max_"+comp_id):

			print("valid entry for comp_id: ",str(comp_id))

			# get component name for Lite-PI
			sql = "SELECT LitePiComponentName,ComponentName FROM ComponentType WHERE ComponentID = %s"
			val = (comp_id,)
			rs_comp_name = execute_query(sql,val)

			if rs_comp_name != ():
				comp_name = rs_comp_name[0][0]
			else:
				comp_name = "Comp"+str(random.randint(1,100))

			# get pre-defined json file format
			result = get_litepi_output_json_file_format()

			# Board level details
			if request.form.get("quick_ir") == "yes":
				result["isDCRBoard"] = "true"						# Quick IR for Board
				save_litepi_project["is_quick_ir_selected"] = True
			else:
				result["isDCRBoard"] = "false"						# Quick IR for Board

			result["isDCRPackage"] = "false"						# Quick IR for Package
			result["isPackageDesignType"] = "false"					# Board - false. Package - true
			result["projDir"] = os.path.join(lsf_path,"Board_ID_"+str(boardid),rev,comp_name)
			result["projName"] = "LitePiBoardID"+str(boardid)

			# advanceSetting
			result["advanceSetting"]["capPrefix"] = request.form.get("cap_prefix",type=str)
			result["advanceSetting"]["CompDepth"] = request.form.get("comp_depth",type=str)
			result["advanceSetting"]["CpuUsage"] = request.form.get("parallel_launch",type=str)
			result["advanceSetting"]["IgnoreNetKeyWordList"] = request.form.get("brd_ignore",type=str)
			result["advanceSetting"]["InductorPrefix"] = request.form.get("ind_prefix",type=str)

			if request.form.get("gen_data_charts") is None:
				result["advanceSetting"]["isGenerate3DPlots"] = "false"
			else:
				result["advanceSetting"]["isGenerate3DPlots"] = "true"

			if request.form.get("xy_location") is None:
				result["advanceSetting"]["isXY"] = "false"
			else:
				result["advanceSetting"]["isXY"] = "true"

			result["advanceSetting"]["max"] = request.form.get("max",type=str)
			result["advanceSetting"]["mean"] = request.form.get("mean",type=str)
			result["advanceSetting"]["min"] = request.form.get("min",type=str)
			result["advanceSetting"]["multiplier"] = request.form.get("sec_large",type=str)
			result["advanceSetting"]["passing"] = request.form.get("passing",type=str)
			result["advanceSetting"]["resistorPrefix"] = request.form.get("res_prefix",type=str)
			result["advanceSetting"]["stdDev"] = request.form.get("std_deviation",type=str)


			#dbInfoNew
			result["dbInfoNew"]["dbInputFilename"] = os.path.join(lsf_path,"Board_ID_"+str(boardid),rev,comp_name,"new_spd_file.spd")
			result["dbInfoNew"]["dbName"] = "NewBoard"
			result["dbInfoNew"]["dbTCL"] = os.path.join(lsf_path,"Board_ID_"+str(boardid),rev,comp_name,result["dbInfoNew"]["dbName"]+"_TclTemplate.tcl")
			
			if request.form.get("ref_des_1"):
				result["dbInfoNew"]["dbVrm"] = request.form.get("ref_des_1",type=str)

			if request.form.get("gnd_net_1"):
				result["dbInfoNew"]["gndNet"] = request.form.get("gnd_net_1",type=str)

			#result["dbInfoNew"]["selectedDieList"] = []


			# dbInfoReference
			#if boardid_ref:
			if ref_board_name != "":
				result["dbInfoReference"]["dbInputFilename"] = os.path.join(lsf_path,"Board_ID_"+str(boardid),rev,comp_name,"ref_spd_file.spd")
				result["dbInfoReference"]["dbName"] = "RefBoard"
				result["dbInfoReference"]["dbTCL"] = os.path.join(lsf_path,"Board_ID_"+str(boardid),rev,comp_name,result["dbInfoReference"]["dbName"]+"_TclTemplate.tcl")

				if request.form.get("ref_des_2"):
					result["dbInfoReference"]["dbVrm"] = request.form.get("ref_des_2",type=str)

				if request.form.get("gnd_net_2"):
					result["dbInfoReference"]["gndNet"] = request.form.get("gnd_net_2",type=str)

				#result["dbInfoReference"]["selectedDieList"] = []

			# get stackup details
			stackup_details = get_stackup_details_json(boardid=boardid,file_id=new_file_upload_id)

			# stackupListNewDB
			result = update_stackup_details_in_json(key=stackup_list_new_db,json_file=result,stackup_data=stackup_details)

			# stackupListReferenceDB
			#if boardid_ref:
			if ref_board_name != "":

				# get stackup details
				stackup_details = get_stackup_details_json(boardid=boardid_ref,file_id=ref_file_upload_id)

				result = update_stackup_details_in_json(key=stackup_list_ref_db,json_file=result,stackup_data=stackup_details)

			# update path delimiter from "/" to "\\"
			result["projDir"] = result["projDir"].replace("/","\\")
			result["dbInfoNew"]["dbInputFilename"] = result["dbInfoNew"]["dbInputFilename"].replace("/","\\")
			result["dbInfoNew"]["dbTCL"] = result["dbInfoNew"]["dbTCL"].replace("/","\\")
			result["dbInfoReference"]["dbInputFilename"] = result["dbInfoReference"]["dbInputFilename"].replace("/","\\")
			result["dbInfoReference"]["dbTCL"] = result["dbInfoReference"]["dbTCL"].replace("/","\\")

			# get pre-defined dict for simulation list
			temp = get_pre_defined_simulation_list()

			temp["iccMax"] = request.form.get("icc_max_"+comp_id,type=str)
			temp["name"] = copy.deepcopy(comp_name)
			temp["net1"] = request.form.get("new_rail_name_"+comp_id,type=str)
			temp["net1VrmList"] = request.form.getlist("new_vrm_list_"+comp_id)

			# if reference design available
			if request.form.get("ref_rail_name_"+comp_id):
				temp["net2"] = request.form.get("ref_rail_name_"+comp_id,type=str)
				temp["net2VrmList"] = request.form.getlist("ref_vrm_list_"+comp_id)

			save_litepi_project["comp_configs"][comp_id] = {}
			save_litepi_project["comp_configs"][comp_id]["icc_max"] = copy.deepcopy(temp["iccMax"])
			save_litepi_project["comp_configs"][comp_id]["interface_name"] = copy.deepcopy(temp["name"])
			save_litepi_project["comp_configs"][comp_id]["new_rail_name"] = copy.deepcopy(temp["net1"])
			save_litepi_project["comp_configs"][comp_id]["new_vrm_ref_des"] = copy.deepcopy(temp["net1VrmList"])
			save_litepi_project["comp_configs"][comp_id]["ref_rail_name"] = copy.deepcopy(temp["net2"])
			save_litepi_project["comp_configs"][comp_id]["ref_vrm_ref_des"] = copy.deepcopy(temp["net2VrmList"])

			result["simulationList"].append(temp)
			all_simulation_details.append(temp)

			# write and push into server for trigger the run
			json_file_base = os.path.join(cloud_base_url,"litepi","temp","Board_ID_"+str(boardid),rev,comp_name)
			json_file_path = os.path.join(json_file_base,"config_file.json")

			if not os.path.exists(json_file_base):
				os.makedirs(json_file_base)

			try:
				with open(json_file_path,"w") as f:
					f.write(json.dumps(result,sort_keys=True,indent=4))
					f.close()

				print("success on writing json file in local.")
			except Exception as inst:
				if is_logging:
					logging.exception('')
				print("Error writting file: ",inst)

			if sr_no==1:

				if boardid_ref in [0,"0","",None]:
					boardid_ref = 0

				# updating DB for tracking purpose at server machine
				sql = "INSERT INTO LitePiRunDetails (BoardID,RunNo,NewFileID,RefFileID,IsRunning,Status,TriggeredOn,TriggeredBy,RefBoardID,NewBoardName,RefBoardName,IsEmailTriggered) VALUES(%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"
				val = (boardid,run_no,new_file_upload_id,ref_file_upload_id,"no","Pending",t,wwid,boardid_ref,new_board_name,ref_board_name,"no")
				execute_query(sql,val)

			# add entry in LitePiRunInterfaceLevelDetails table for tracking
			sql = "INSERT INTO LitePiRunInterfaceLevelDetails (BoardID,RunNo,ComponentID,IsRunning,IsReportsFetchTriggered,Status,IsTriggered,IsTriggerPicked) VALUES(%s,%s,%s,%s,%s,%s,%s,%s)"
			val = (boardid,run_no,comp_id,"no","no","Pending","no","no")
			execute_query(sql,val)

			sr_no+=1

	# clearing simulation list for writing summary data in summary file
	result["simulationList"] = all_simulation_details

	if not os.path.exists(os.path.join(cloud_base_url,"litepi","temp","Board_ID_"+str(boardid),rev)):
		os.makedirs(os.path.join(cloud_base_url,"litepi","temp","Board_ID_"+str(boardid),rev))

	simulation_json_file_path = os.path.join(cloud_base_url,"litepi","temp","Board_ID_"+str(boardid),rev,"SimulationList.json")

	try:
		with open(simulation_json_file_path,"w") as f:
			f.write(json.dumps(result,sort_keys=True,indent=4))
			f.close()

		print("success on writing summary json file in local.")
	except Exception as inst:
		if is_logging:
			logging.exception('')
		print("Error writting summary json file: ",inst)

	# creating input details file
	input_details_file_path = os.path.join(cloud_base_url,"litepi","temp","Board_ID_"+str(boardid),rev,"Summary.xlsx")
	simulation_json_file_path = os.path.join(cloud_base_url,"litepi","temp","Board_ID_"+str(boardid),rev,"SimulationList.json")
	create_litepi_input_details(input_json_file_path=simulation_json_file_path,input_details_file_path=input_details_file_path,boardid=boardid,run_no=run_no,comp_name="")

	summary_base_file_path = os.path.join(cloud_base_url,"litepi","reports","Board_ID_"+str(boardid),rev)
	summary_details_file_path = os.path.join(summary_base_file_path,"Summary.xlsx")

	if not os.path.exists(summary_base_file_path):
		os.makedirs(summary_base_file_path)

	shutil.copy(input_details_file_path,summary_details_file_path)

	'''
	sql = "INSERT INTO LitePiRunFileMappings (BoardID,RunNo,NewPiFileID,RefPiFileID,NewFileID,RefFileID) VALUES(%s,%s,%s,%s,%s,%s)"
	val = (boardid,run_no,new_file_upload_id,ref_file_upload_id,None,None)
	execute_query(sql,val)
	'''

	# Save project
	save_litepi_project["new_design"]["design_id"] = boardid
	save_litepi_project["new_design"]["design_name"] = new_board_name
	save_litepi_project["new_design"]["file_id"] = new_file_upload_id
	save_litepi_project["new_design"]["gnd_net"] = copy.deepcopy(result["dbInfoNew"]["gndNet"])
	save_litepi_project["new_design"]["soc_ref_des"] = copy.deepcopy(result["dbInfoNew"]["dbVrm"])


	if ref_design_selection == "new":
		save_litepi_project["ref_design"]["is_new_design"] = True
	else:
		save_litepi_project["ref_design"]["is_new_design"] = False

	save_litepi_project["ref_design"]["design_id"] = boardid_ref
	save_litepi_project["ref_design"]["design_name"] = ref_board_name
	save_litepi_project["ref_design"]["file_id"] = ref_file_upload_id
	save_litepi_project["ref_design"]["gnd_net"] = copy.deepcopy(result["dbInfoReference"]["gndNet"])
	save_litepi_project["ref_design"]["soc_ref_des"] = copy.deepcopy(result["dbInfoReference"]["dbVrm"])

	save_litepi_project["advanceSetting"] = result["advanceSetting"]
	save_litepi_project["comp_selected_list"] = comp_selected_list

	#print("save_litepi_project: ",save_litepi_project)
	try:
		sql = "INSERT INTO LitePiProjects(BoardID,RunNo,ConfigJson) VALUES(%s,%s,%s)"
		val = (boardid,run_no,str(json.dumps(save_litepi_project,sort_keys=True,indent=0)))
		execute_query(sql,val)

	except Exception as inst:
		print(inst)

	return render("lite_pi_success.html",username=username,user_role_name=user_role_name,region_name=region_name,is_admin=is_admin,is_elec_owner=is_elec_owner,is_design_owner=is_design_owner,is_layout_owner=is_layout_owner)

def update_stackup_details_in_json(key="",json_file={},stackup_data=[]):

	'''
	Key:
		1. stackup_list_new_db = "stackupListNewDB"
		2. stackup_list_ref_db = "stackupListReferenceDB"

	'''
	thickness = "Thickness"
	conductivity = "Conductivity"
	permittivity = "Permittivity"
	loss_tangent = "LossTangent"
	layer_name = "LayerName"
	layer_type = "Type"

	if key in json_file:

		for row in stackup_data:

			temp = {}
			temp[layer_name] = str(row[1])
			temp[layer_type] = str(row[2])
			temp[thickness] = str(row[3])
			temp[conductivity] = str(row[4])
			temp[permittivity] = str(row[5])
			temp[loss_tangent] = str(row[6])

			json_file[key].append(temp)

	return json_file

def get_litepi_output_json_file_format():
	'''
	output json file in pre-defined format with default values, after pulling it user can modify the data on top of this
	'''
	result = {}

	# Board level details
	result["extractIM"] = r"C:\Cadence\Sigrity2021.1\tools\bin\XtractIM.exe"
	result["isDCRBoard"] = "false"
	result["isDCRPackage"] = "false"
	result["isPackageDesignType"] = "false"
	result["projDir"] = ""
	result["projName"] = ""
	result["advanceSetting"] = {}
	result["dbInfoNew"] = {}
	result["stackupListNewDB"] = []
	result["dbInfoReference"] = {}
	result["stackupListReferenceDB"] = []
	result["simulationList"] = []

	# advanceSetting
	result["advanceSetting"]["capPrefix"] = "C"
	result["advanceSetting"]["CompDepth"] = "4"
	result["advanceSetting"]["CpuUsage"] = "2"
	result["advanceSetting"]["IgnoreNetKeyWordList"] = "VAL"
	result["advanceSetting"]["InductorPrefix"] = "L"
	result["advanceSetting"]["isGenerate3DPlots"] = "false"
	result["advanceSetting"]["isXY"] = "true"
	result["advanceSetting"]["max"] = "5"
	result["advanceSetting"]["mean"] = "5"
	result["advanceSetting"]["min"] = "-5"
	result["advanceSetting"]["multiplier"] = "10"
	result["advanceSetting"]["passing"] = "0"
	result["advanceSetting"]["resistorPrefix"] = "R"
	result["advanceSetting"]["stdDev"] = "5"

	#dbInfoNew
	result["dbInfoNew"]["dbInputFilename"] = ""
	result["dbInfoNew"]["dbName"] = ""
	result["dbInfoNew"]["dbTCL"] = ""
	result["dbInfoNew"]["dbVrm"] = ""
	result["dbInfoNew"]["gndNet"] = ""
	result["dbInfoNew"]["selectedDieList"] = []

	# dbInfoReference
	result["dbInfoReference"]["dbInputFilename"] = ""
	result["dbInfoReference"]["dbName"] = ""
	result["dbInfoReference"]["dbTCL"] = ""
	result["dbInfoReference"]["dbVrm"] = ""
	result["dbInfoReference"]["gndNet"] = ""
	result["dbInfoReference"]["selectedDieList"] = []

	return result

def get_pre_defined_simulation_list():

	result = {}
	result["iccMax"] = 1
	result["name"] = ""
	result["net1"] = ""
	result["net1VrmList"] = []
	result["net2"] = ""
	result["net2VrmList"] = []

	return result

@app.route("/update_stackup_from_user",methods = ['POST', 'GET'])
def update_stackup_from_user():

	wwid = session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	if request.method == "POST":
		boardid = request.form.get("boardid",type=str)
		file_id = request.form.get("file_id",type=str)

		sql = "SELECT BoardID,SequenceNo,StackupName FROM LitePiStackupDetails WHERE BoardID = %s AND FileID = %s ORDER BY SequenceNo"
		val = (boardid,file_id)
		rs = execute_query(sql,val)

		for i in range(len(rs)):

			if request.form.get("stackup_thickness_"+str(i)):

				thickness = request.form.get("stackup_thickness_"+str(i))
				conductivity = request.form.get("stackup_conductivity_"+str(i))
				permittivity = request.form.get("stackup_permittivity_"+str(i))
				loss_tangent = request.form.get("stackup_loss_tangent_"+str(i))

				sql = "UPDATE LitePiStackupDetails SET Thickness = %s, Conductivity = %s, Permittivity = %s, LossTangent = %s, UpdatedOn = %s, UpdatedBy = %s WHERE BoardID = %s AND FileID = %s AND SequenceNo = %s AND StackupName = %s"
				val = (thickness,conductivity,permittivity,loss_tangent,t,wwid,boardid,file_id,rs[i][1],rs[i][2])
				execute_query(sql,val)

	return jsonify(True)

#border to the cells
def set_border(ws, cell_range):
	thin = Side(border_style="thin", color="000000")
	for row in ws[cell_range]:
		for cell in row:
			cell.border = Border(top=thin, left=thin, right=thin, bottom=thin)

#styling the worksheet : adjusting the size of the cell
def adjust_column_width_from_col(ws, min_row, min_col, max_col):
	column_widths = []

	for i, col in enumerate(ws.iter_cols(min_col=min_col, max_col=max_col, min_row=min_row)):

		for cell in col:
			value = cell.value
			if value is not None:
				if isinstance(value, str) is False:
					value = str(value)

				try:
					column_widths[i] = max(column_widths[i], len(value))
				except IndexError:
					column_widths.append(len(value))

	for i, width in enumerate(column_widths):
		col_name = get_column_letter(min_col + i)
		value = column_widths[i] + 2
		ws.column_dimensions[col_name].width = value

	return ws

def create_litepi_input_details(input_json_file_path="",input_details_file_path="",boardid=0, run_no=0, comp_name=""):
	stackup_list_new_db = "stackupListNewDB"
	stackup_list_ref_db = "stackupListReferenceDB"

	stackup_details = {}

	try:
		with open(input_json_file_path, "r") as json_file:
			json_data = json_file.read()
			stackup_details = json.loads(json_data)
			json_file.close()
	except Exception as inst:
		print(inst)
		return False

	print("stackup_details: ",stackup_details)
	# variables
	input_boardid = str(boardid)
	input_ref_boardid = "-"
	input_design_name = "-"
	input_ref_design_name = "-"
	input_file_name = "-"
	input_ref_file_name = "-"

	input_interface_name = comp_name

	if stackup_details["isDCRBoard"] == "true":
		input_quick_ir_drop = "yes"
	else:
		input_quick_ir_drop = "no"

	input_icc_max = stackup_details["simulationList"][0]["iccMax"]
	input_gnd_net = stackup_details["dbInfoNew"]["gndNet"]
	input_ref_gnd_net = stackup_details["dbInfoReference"]["gndNet"]
	input_ref_des = stackup_details["dbInfoNew"]["dbVrm"]
	input_ref_ref_des = stackup_details["dbInfoReference"]["dbVrm"]
	input_rail_name = stackup_details["simulationList"][0]["net1"]
	input_ref_rail_name = stackup_details["simulationList"][0]["net2"]
	input_vrm_ref_des = ','.join(stackup_details["simulationList"][0]["net1VrmList"])
	input_ref_vrm_ref_des = ','.join(stackup_details["simulationList"][0]["net2VrmList"])


	sql = "SELECT a.BoardID,IF(a.RefBoardID>0,a.RefBoardID,''),a.RunNo,a.NewBoardName,IFNULL(a.RefBoardName,''),IFNULL(b.FileName,''),IFNULL(c.FileName,'') FROM LitePiRunDetails a LEFT JOIN LitePiFileStorage b ON a.NewFileID = b.FileID LEFT JOIN LitePiFileStorage c ON a.RefFileID = c.FileID WHERE a.BoardID = %s AND a.RunNo = %s"
	val = (boardid, run_no)
	rs_input = execute_query(sql, val)

	if rs_input != ():
		input_ref_boardid = str(rs_input[0][1])
		input_design_name = str(rs_input[0][3])
		input_ref_design_name = str(rs_input[0][4])
		input_file_name = str(rs_input[0][5])
		input_ref_file_name = str(rs_input[0][6])


	try:
		wb = openpyxl.Workbook()
		wb.save(input_details_file_path)
	except Exception as inst:
		print("Error creating input details file. Error: ", inst)
		return False

	#InputDetails Sheet
	try:
		wb = openpyxl.load_workbook(filename=input_details_file_path)
		ws1 = wb.create_sheet('Input Details')
		ws1.title = "Input Details"
	except Exception as inst:
		print(inst)
		return False


	try:

		#First table
		for rows in ws1.iter_rows(min_row=1, max_row=1, min_col=1, max_col=3):
			for cell in rows:
				cell.fill = PatternFill(start_color=firstHeader, end_color=firstHeader, fill_type="solid")
		ws1.cell(1,2).value = "New Design"
		ws1.cell(1, 2).font = Font(bold=True)
		ws1.cell(1, 2).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.cell(1,3).value = "Reference Design"
		ws1.cell(1, 3).font = Font(bold=True)
		ws1.cell(1, 3).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.cell(2,1).value = "Quick IR Drop"
		ws1.cell(2, 1).font = Font(bold=True)
		ws1.cell(2, 1).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.merge_cells('B2:C2')
		ws1.cell(2,2).value = input_quick_ir_drop
		ws1.cell(2, 2).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.merge_cells('A3:C3')

		ws1.merge_cells('A4:C4')
		for rows in ws1.iter_rows(min_row=4, max_row=4, min_col=1, max_col=3):
			for cell in rows:
				cell.fill = PatternFill(start_color=secondHeader, end_color=secondHeader, fill_type="solid")
		ws1.cell(4,1).value = "Design Details"
		ws1.cell(4, 1).font = Font(bold=True)
		ws1.cell(4, 1).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.cell(5,1).value = "Design ID"
		ws1.cell(5, 1).font = Font(bold=True)
		ws1.cell(5, 1).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.cell(5,2).value = input_boardid
		ws1.cell(5, 2).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)
		ws1.cell(5,3).value = input_ref_boardid
		ws1.cell(5, 3).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)


		ws1.cell(6, 1).value = "Design Name"
		ws1.cell(6, 1).font = Font(bold=True)
		ws1.cell(6, 1).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.cell(6, 2).value = input_design_name
		ws1.cell(6, 2).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)
		ws1.cell(6, 3).value = input_ref_design_name
		ws1.cell(6, 3).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.cell(7, 1).value = "File Name"
		ws1.cell(7, 1).font = Font(bold=True)
		ws1.cell(7, 1).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.cell(7, 2).value = input_file_name
		ws1.cell(7, 2).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)
		ws1.cell(7, 3).value = input_ref_file_name
		ws1.cell(7, 3).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.merge_cells('A8:C8')

		ws1.merge_cells('A9:C9')
		for rows in ws1.iter_rows(min_row=9, max_row=9, min_col=1, max_col=3):
			for cell in rows:
				cell.fill = PatternFill(start_color=secondHeader, end_color=secondHeader, fill_type="solid")
		ws1.cell(9, 1).value = "Soc Section"
		ws1.cell(9, 1).font = Font(bold=True)
		ws1.cell(9, 1).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.cell(10, 1).value = "GND Net"
		ws1.cell(10, 1).font = Font(bold=True)
		ws1.cell(10, 1).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.cell(10, 2).value = input_gnd_net
		ws1.cell(10, 2).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)
		ws1.cell(10, 3).value = input_ref_gnd_net
		ws1.cell(10, 3).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		ws1.cell(11, 1).value = "Ref Des"
		ws1.cell(11, 1).font = Font(bold=True)
		ws1.cell(11, 1).alignment = Alignment(horizontal=horizontal,
											  vertical=vertical)
		ws1.cell(11, 2).value = input_ref_des
		ws1.cell(11, 2).alignment = Alignment(horizontal=horizontal,
											  vertical=vertical)
		ws1.cell(11, 3).value = input_ref_ref_des
		ws1.cell(11, 3).alignment = Alignment(horizontal=horizontal,
											  vertical=vertical)
		set_border(ws1,'A1:C11')



		#Second table - Advanced Settings:
		ws1.merge_cells('A15:B15')
		for rows in ws1.iter_rows(min_row=15, max_row=15, min_col=1, max_col=2):
			for cell in rows:
				cell.fill = PatternFill(start_color=firstHeader, end_color=firstHeader, fill_type="solid")
		ws1.cell(15,1).value = "Advanced Settings"
		ws1.cell(15, 1).font = Font(bold=True)
		ws1.cell(15, 1).alignment = Alignment(horizontal=horizontal,
											  vertical=vertical)


		ws1.cell(16,1).value = "Max. (%)"
		ws1.cell(16,2).value = stackup_details["advanceSetting"]["max"]

		ws1.cell(17,1).value = "Min. (%)"
		ws1.cell(17,2).value = stackup_details["advanceSetting"]["min"]

		ws1.cell(18,1).value = "Mean (%)"
		ws1.cell(18,2).value = stackup_details["advanceSetting"]["mean"]

		ws1.cell(19,1).value = "Standard Deviation (%)"
		ws1.cell(19,2).value = stackup_details["advanceSetting"]["stdDev"]

		ws1.cell(20,1).value = "Passing"
		ws1.cell(20,2).value = stackup_details["advanceSetting"]["passing"]

		ws1.cell(21,1).value = "Second Largest value multiplier to remove Largest"
		ws1.cell(21,2).value = stackup_details["advanceSetting"]["multiplier"]

		ws1.cell(22,1).value = "Component depth"
		ws1.cell(22,2).value = stackup_details["advanceSetting"]["CompDepth"]

		ws1.cell(23,1).value = "Generate Data Charts by Bump XY locations"
		if stackup_details["advanceSetting"]["isGenerate3DPlots"] == "true":
			ws1.cell(23, 2).value = "yes"
		else:
			ws1.cell(23,2).value = "no"

		ws1.cell(24,1).value = "is XY locations"
		if stackup_details["advanceSetting"]["isXY"] == "true":
			ws1.cell(24, 2).value = "yes"
		else:
			ws1.cell(24,2).value = "no"

		ws1.cell(25,1).value = "Board ignore net key words"
		ws1.cell(25,2).value = stackup_details["advanceSetting"]["IgnoreNetKeyWordList"]

		ws1.cell(26,1).value = "Capacitor prefix (i.e: C, PC)"
		ws1.cell(26,2).value = stackup_details["advanceSetting"]["capPrefix"]

		ws1.cell(27,1).value = "Resistor prefix (i.e: R, PR)"
		ws1.cell(27,2).value = stackup_details["advanceSetting"]["resistorPrefix"]

		ws1.cell(28,1).value = "Inductor prefix (i.e: L, PL)"
		ws1.cell(28,2).value = stackup_details["advanceSetting"]["InductorPrefix"]

		set_border(ws1, 'A15:B28')


		#Third table : Interfaces Selected for Review
		ws1.merge_cells('F1:K1')
		for rows in ws1.iter_rows(min_row=1, max_row=1, min_col=6, max_col=11):
			for cell in rows:
				cell.fill = PatternFill(start_color=firstHeader, end_color=firstHeader, fill_type="solid")
		ws1.cell(1,6).value = "Interfaces Selected for Review"
		ws1.cell(1,6).font = Font(bold=True)
		ws1.cell(1,6).alignment = Alignment(horizontal=horizontal,
											  vertical=vertical)

		ws1.merge_cells('F2:G2')
		for rows in ws1.iter_rows(min_row=2, max_row=2, min_col=6, max_col=11):
			for cell in rows:
				cell.fill = PatternFill(start_color=secondHeader, end_color=secondHeader, fill_type="solid")
		ws1.merge_cells('H2:I2')
		ws1.cell(2,8).value = "New Design"
		ws1.cell(2,8).font = Font(bold=True)
		ws1.cell(2,8).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)
		ws1.merge_cells('J2:K2')
		ws1.cell(2,10).value = "Reference Design"
		ws1.cell(2,10).font = Font(bold=True)
		ws1.cell(2,10).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		for rows in ws1.iter_rows(min_row=3, max_row=3, min_col=6, max_col=11):
			for cell in rows:
				cell.fill = PatternFill(start_color=secondHeader, end_color=secondHeader, fill_type="solid")
		ws1.cell(3,6).value = "Interfaces"
		ws1.cell(3,6).font = Font(bold=True)
		ws1.cell(3,6).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)
		ws1.cell(3,7).value = "Icc Max"
		ws1.cell(3,7).font = Font(bold=True)
		ws1.cell(3,7).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)
		ws1.cell(3,8).value = "Rail Name"
		ws1.cell(3,8).font = Font(bold=True)
		ws1.cell(3,8).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)
		ws1.cell(3,9).value = "Vrm Ref Des"
		ws1.cell(3, 9).font = Font(bold=True)
		ws1.cell(3, 9).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)
		ws1.cell(3, 10).value = "Rail Name"
		ws1.cell(3, 10).font = Font(bold=True)
		ws1.cell(3, 10).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)
		ws1.cell(3, 11).value = "Vrm Ref Des"
		ws1.cell(3, 11).font = Font(bold=True)
		ws1.cell(3, 11).alignment = Alignment(horizontal=horizontal,
											  vertical=vertical)
		set_border(ws1, 'F1:K3')

		simulationList = stackup_details['simulationList']

		#print(simulationList)
		row = 4
		for s in simulationList:
			ws1.cell(row = row,column=6).value = s['name']
			ws1.cell(row=row, column=7).value = s['iccMax']
			ws1.cell(row=row, column=8).value = s['net1']
			ws1.cell(row=row, column=9).value = ','.join(s['net1VrmList'])
			ws1.cell(row=row, column=10).value = s['net2']
			ws1.cell(row=row, column=11).value = ','.join(s['net2VrmList'])
			row+=1
		rowValue = row
		ws = adjust_column_width_from_col(ws1, 1, 1, ws1.max_column)

		thin = Border(left=Side(border_style="thin", color="000000"),
					  right=Side(border_style="thin", color="000000"),
					  top=Side(border_style="thin", color="000000"),
					  bottom=Side(border_style="thin", color="000000"))

		for x in range(4, rowValue):
			for y in range(6, 12):
				ws1.cell(row=x, column=y).border = thin
				if x == 4:
					ws1.cell(row=x, column=y).border = thin
				if x == 4 + rowValue - 1:
					ws1.cell(row=x, column=y).border = thin


		wb.save(input_details_file_path)


		# StackupDetails Sheet
		try:
			wb = openpyxl.load_workbook(filename=input_details_file_path)
			ws2 = wb.create_sheet('Stackup Details')
			ws2.title = "Stackup Details"
		except Exception as inst:
			print(inst)
			return False

		# StackUpListNewDB
		lstNewDb = stackup_details['stackupListNewDB']

		ws2.cell(1, 1).value = "New Design"

		ws2.cell(1, 1).style = "Headline 3"
		ws2.cell(1, 1).alignment = Alignment(horizontal=horizontal,
											 vertical=vertical)

		# header values of stackupListNewDb
		ws2.cell(2, 1, "S.No")
		ws2.cell(2, 2, "Layer Name")
		ws2.cell(2, 3, "Layer Type")
		ws2.cell(2, 4, "Thickness (µm)")
		ws2.cell(2, 5, "Conductivity")
		ws2.cell(2, 6, "Permittivity")
		ws2.cell(2, 7, "Loss Tangent")

		# values
		lst1 = []
		row = 3
		sno = 1
		for val in lstNewDb:
			ws2.cell(row=row, column=1).value = sno
			ws2.cell(row=row, column=2).value = val["LayerName"]
			ws2.cell(row=row, column=3).value = val["Type"]
			ws2.cell(row=row, column=4).value = val["Thickness"]
			ws2.cell(row=row, column=5).value = val["Conductivity"]
			ws2.cell(row=row, column=6).value = val["Permittivity"]
			ws2.cell(row=row, column=7).value = val["LossTangent"]

			sno += 1
			row += 1

		maxRowNewDB = row
		maxColumnNewDb = 8

		# StackupListReferenceDB
		lstRefDb = stackup_details['stackupListReferenceDB']

		ws2.cell(1, 10).value = "Reference Design"

		ws2.cell(1, 10).style = "Headline 3"
		ws2.cell(1, 10).alignment = Alignment(horizontal="center",
											  vertical="center")

		# header values of stackupListNewDb
		ws2.cell(2, 10, "S.No")
		ws2.cell(2, 11, "Layer Name")
		ws2.cell(2, 12, "Layer Type")
		ws2.cell(2, 13, "Thickness (µm)")
		ws2.cell(2, 14, "Conductivity")
		ws2.cell(2, 15, "Permittivity")
		ws2.cell(2, 16, "Loss Tangent")

		# values
		row = 3
		sno = 1
		for val in lstRefDb:
			ws2.cell(row=row, column=10).value = sno
			ws2.cell(row=row, column=11).value = val["LayerName"]
			ws2.cell(row=row, column=12).value = val["Type"]
			ws2.cell(row=row, column=13).value = val["Thickness"]
			ws2.cell(row=row, column=14).value = val["Conductivity"]
			ws2.cell(row=row, column=15).value = val["Permittivity"]
			ws2.cell(row=row, column=16).value = val["LossTangent"]

			sno += 1
			row += 1

		maxRowRefDB = row
		maxColumnRefDb = 17

		# styling
		# merging
		min_Newrow = 1
		max_Newrow = maxRowNewDB - 1
		min_Newcol = 1
		max_Newcol = maxColumnNewDb - 1

		ws2.merge_cells(start_row=min_Newrow, start_column=min_Newcol, end_row=min_Newrow, end_column=max_Newcol)

		min_Refrow = 1
		max_Refrow = maxRowRefDB - 1
		min_Refcol = 10
		max_Refcol = maxColumnRefDb - 1

		ws2.merge_cells(start_row=min_Refrow, start_column=min_Refcol, end_row=min_Refrow, end_column=max_Refcol)

		# setting the dimension
		dim_holder = DimensionHolder(worksheet=ws2)

		for col in range(ws2.min_column, ws2.max_column + 1):
			dim_holder[get_column_letter(col)] = ColumnDimension(ws2, min=col, max=col, width=10.5)

		ws2.column_dimensions = dim_holder

		# Filling the topmost heading with yellow color
		for rows in ws2.iter_rows(min_row=min_Newrow, max_row=min_Newrow, min_col=min_Newcol, max_col=max_Newcol):
			for cell in rows:
				if cell.row % 2:
					cell.fill = PatternFill(start_color=firstHeader, end_color=firstHeader, fill_type="solid")


		for rows in ws2.iter_rows(min_row=min_Refrow, max_row=min_Refrow, min_col=min_Refcol, max_col=max_Refcol):
			for cell in rows:
				if cell.row % 2:
					cell.fill = PatternFill(start_color=firstHeader, end_color=firstHeader, fill_type="solid")

		fill_cell = PatternFill(start_color=secondHeader, end_color=secondHeader, fill_type='solid')

		thin = Border(left=Side(border_style="thin", color="000000"),
					  right=Side(border_style="thin", color="000000"),
					  top=Side(border_style="thin", color="000000"),
					  bottom=Side(border_style="thin", color="000000"))

		for x in range(min_Newrow, max_Newrow + 1):
			for y in range(min_Newcol, max_Newcol + 1):
				ws2.cell(row=x, column=y).border = thin
				if x == min_Newrow:
					ws2.cell(row=x, column=y).border = thin
					ws2.cell(row=x + 1, column=y).fill = fill_cell
				if x == min_Newrow + max_Newrow - 1:
					ws2.cell(row=x, column=y).border = thin

		for x in range(min_Refrow, max_Refrow + 1):
			for y in range(min_Refcol, max_Refcol + 1):
				ws2.cell(row=x, column=y).border = thin
				if x == min_Refrow:
					ws2.cell(row=x, column=y).border = thin
					ws2.cell(row=x + 1, column=y).fill = fill_cell
				if x == min_Refrow + max_Refrow - 1:
					ws2.cell(row=x, column=y).border = thin

		adjust_column_width_from_col(ws2, 2, min_Refcol, max_Refcol)
		adjust_column_width_from_col(ws2, 2, min_Newcol, max_Newcol)


		wb.save(input_details_file_path)
	
	except Exception as inst:
		print(inst)
		return False

	return True

def create_final_summary_file(rev_path=""):
    print("create_final_summary_file...")
    print("rev_path: ", rev_path)

    try:
        # creating the workSheet Summary
        print("welcome")

    except Exception as inst:
        print("Error in writing summary file. Error: ", inst)

    if True:
        excelFile = os.path.join(rev_path, "Summary.xlsx")


        #wb = openpyxl.Workbook()
        #wb.save(excelFile)

        wb = openpyxl.load_workbook(filename=excelFile)
        ws = wb.active
        ws.title = "Summary"

        # styling the Summary worksheet:
        # styling row1
        ws.merge_cells('A1:A4')
        ws['A1'].value = "FINAL RESULT"
        ws['A1'].font = Font(bold=True)
        ws['A1'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('B1:B4')
        ws['B1'].value = "POWER RAIL"
        ws['B1'].font = Font(bold=True)
        ws['B1'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('C1:N1')
        ws['C1'].value = "VRM"
        ws['C1'].font = Font(bold=True)
        ws['C1'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('O1:Z1')
        ws['O1'].value = "PKG Side"
        ws['O1'].font = Font(bold=True)
        ws['O1'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('AA1:AL1')
        ws['AA1'].value = "CAP"
        ws['AA1'].font = Font(bold=True)
        ws['AA1'].alignment = Alignment(horizontal="center", vertical="center")

        # styling row2
        ws.merge_cells('C2:H2')
        ws['C2'].value = "Rdc"
        ws['C2'].font = Font(bold=True)
        ws['C2'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('I2:N2')
        ws['I2'].value = "L-self"
        ws['I2'].font = Font(bold=True)
        ws['I2'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('O2:T2')
        ws['O2'].value = "Rdc"
        ws['O2'].font = Font(bold=True)
        ws['O2'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('U2:Z2')
        ws['U2'].value = "L-self"
        ws['U2'].font = Font(bold=True)
        ws['U2'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('AA2:AF2')
        ws['AA2'].value = "Rdc"
        ws['AA2'].font = Font(bold=True)
        ws['AA2'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('AG2:AL2')
        ws['AG2'].value = "L-self"
        ws['AG2'].font = Font(bold=True)
        ws['AG2'].alignment = Alignment(horizontal="center", vertical="center")

        # styling row3
        ws.merge_cells('C3:D3')
        ws['C3'].value = "MAX"
        ws['C3'].font = Font(bold=True)
        ws['C3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('E3:F3')
        ws['E3'].value = "MIN"
        ws['E3'].font = Font(bold=True)
        ws['E3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('G3:H3')
        ws['G3'].value = "MEAN"
        ws['G3'].font = Font(bold=True)
        ws['G3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('I3:J3')
        ws['I3'].value = "MAX"
        ws['I3'].font = Font(bold=True)
        ws['I3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('K3:L3')
        ws['K3'].value = "MIN"
        ws['K3'].font = Font(bold=True)
        ws['K3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('M3:N3')
        ws['M3'].value = "MEAN"
        ws['M3'].font = Font(bold=True)
        ws['M3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('O3:P3')
        ws['O3'].value = "MAX"
        ws['O3'].font = Font(bold=True)
        ws['O3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('Q3:R3')
        ws['Q3'].value = "MIN"
        ws['Q3'].font = Font(bold=True)
        ws['Q3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('S3:T3')
        ws['S3'].value = "MEAN"
        ws['S3'].font = Font(bold=True)
        ws['S3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('U3:V3')
        ws['U3'].value = "MAX"
        ws['U3'].font = Font(bold=True)
        ws['U3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('W3:X3')
        ws['W3'].value = "MIN"
        ws['W3'].font = Font(bold=True)
        ws['W3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('Y3:Z3')
        ws['Y3'].value = "MEAN"
        ws['Y3'].font = Font(bold=True)
        ws['Y3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('AA3:AB3')
        ws['AA3'].value = "MAX"
        ws['AA3'].font = Font(bold=True)
        ws['AA3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('AC3:AD3')
        ws['AC3'].value = "MIN"
        ws['AC3'].font = Font(bold=True)
        ws['AC3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('AE3:AF3')
        ws['AE3'].value = "MEAN"
        ws['AE3'].font = Font(bold=True)
        ws['AE3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('AG3:AH3')
        ws['AG3'].value = "MAX"
        ws['AG3'].font = Font(bold=True)
        ws['AG3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('AI3:AJ3')
        ws['AI3'].value = "MIN"
        ws['AI3'].font = Font(bold=True)
        ws['AI3'].alignment = Alignment(horizontal="center", vertical="center")

        ws.merge_cells('AK3:AL3')
        ws['AK3'].value = "MEAN"
        ws['AK3'].font = Font(bold=True)
        ws['AK3'].alignment = Alignment(horizontal="center", vertical="center")

        # styling row4
        ws['C4'].value = "New"
        ws['C4'].font = Font(bold=True)
        ws['C4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['D4'].value = "Ref"
        ws['D4'].font = Font(bold=True)
        ws['D4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['E4'].value = "New"
        ws['E4'].font = Font(bold=True)
        ws['E4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['F4'].value = "Ref"
        ws['F4'].font = Font(bold=True)
        ws['F4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['G4'].value = "New"
        ws['G4'].font = Font(bold=True)
        ws['G4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['H4'].value = "Ref"
        ws['H4'].font = Font(bold=True)
        ws['H4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['I4'].value = "New"
        ws['I4'].font = Font(bold=True)
        ws['I4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['J4'].value = "Ref"
        ws['J4'].font = Font(bold=True)
        ws['J4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['K4'].value = "New"
        ws['K4'].font = Font(bold=True)
        ws['K4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['L4'].value = "Ref"
        ws['L4'].font = Font(bold=True)
        ws['L4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['M4'].value = "New"
        ws['M4'].font = Font(bold=True)
        ws['M4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['N4'].value = "Ref"
        ws['N4'].font = Font(bold=True)
        ws['N4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['O4'].value = "New"
        ws['O4'].font = Font(bold=True)
        ws['O4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['P4'].value = "Ref"
        ws['P4'].font = Font(bold=True)
        ws['P4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['Q4'].value = "New"
        ws['Q4'].font = Font(bold=True)
        ws['Q4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['R4'].value = "Ref"
        ws['R4'].font = Font(bold=True)
        ws['R4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['S4'].value = "New"
        ws['S4'].font = Font(bold=True)
        ws['S4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['T4'].value = "Ref"
        ws['T4'].font = Font(bold=True)
        ws['T4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['U4'].value = "New"
        ws['U4'].font = Font(bold=True)
        ws['U4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['V4'].value = "Ref"
        ws['V4'].font = Font(bold=True)
        ws['V4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['W4'].value = "New"
        ws['W4'].font = Font(bold=True)
        ws['W4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['X4'].value = "Ref"
        ws['X4'].font = Font(bold=True)
        ws['X4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['Y4'].value = "New"
        ws['Y4'].font = Font(bold=True)
        ws['Y4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['Z4'].value = "Ref"
        ws['Z4'].font = Font(bold=True)
        ws['Z4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['AA4'].value = "New"
        ws['AA4'].font = Font(bold=True)
        ws['AA4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['AB4'].value = "Ref"
        ws['AB4'].font = Font(bold=True)
        ws['AB4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['AC4'].value = "New"
        ws['AC4'].font = Font(bold=True)
        ws['AC4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['AD4'].value = "Ref"
        ws['AD4'].font = Font(bold=True)
        ws['AD4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['AE4'].value = "New"
        ws['AE4'].font = Font(bold=True)
        ws['AE4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['AF4'].value = "Ref"
        ws['AF4'].font = Font(bold=True)
        ws['AF4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['AG4'].value = "New"
        ws['AG4'].font = Font(bold=True)
        ws['AG4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['AH4'].value = "Ref"
        ws['AH4'].font = Font(bold=True)
        ws['AH4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['AI4'].value = "New"
        ws['AI4'].font = Font(bold=True)
        ws['AI4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['AJ4'].value = "Ref"
        ws['AJ4'].font = Font(bold=True)
        ws['AJ4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['AK4'].value = "New"
        ws['AK4'].font = Font(bold=True)
        ws['AK4'].alignment = Alignment(horizontal="center", vertical="center")

        ws['AL4'].value = "Ref"
        ws['AL4'].font = Font(bold=True)
        ws['AL4'].alignment = Alignment(horizontal="center", vertical="center")

        # to check if the *_Summary file is present in the folders
        pattern = r'_Summary.xlsx'

        newList = []

        try:
            for interface_name in os.listdir(rev_path):
            	if "." not in interface_name:
	                for file_name in os.listdir(os.path.join(rev_path, interface_name)):
	                    if file_name.endswith(pattern):
	                        filePath = os.path.join(rev_path, interface_name, file_name)
	                        newList.append(filePath)

        except Exception as inst:
            print("Error reading folder. Error: ", inst)

        print(newList)

        r = 5
        for item in newList:
            file = item
            workBook = openpyxl.load_workbook(file)
            workSheet = workBook.active
            for i in range(1, 39):
                c = workSheet.cell(row=5, column=i)
                ws.cell(row=r, column=i).value = c.value
                ws.cell(row=r, column=1).fill = copy.copy(workSheet.cell(row=5, column=1).fill)
            r += 1

        sheet = adjust_column_width_from_col(ws, 1, 1, 38)

        # applying border
        double = Border(left=Side(border_style="double", color="000000"),
                        right=Side(border_style="double", color="000000"),
                        top=Side(border_style="double", color="000000"),
                        bottom=Side(border_style="double", color="000000"))

        thin = Border(left=Side(border_style="thin", color="000000"),
                      right=Side(border_style="thin", color="000000"),
                      top=Side(border_style="thin", color="000000"),
                      bottom=Side(border_style="thin", color="000000"))

        for x in range(1, r):
            for y in range(1, 38 + 1):
                ws.cell(row=x, column=y).border = thin
                if x == 1:
                    ws.cell(row=x, column=y).border = thin
                if x == 1 + r:
                    ws.cell(row=x, column=y).border = thin

        # coloring the header rows
        clr1 = "ADD8E6"
        for rows in ws.iter_rows(min_row=1, max_row=1, min_col=3, max_col=38):
            for cell in rows:
                cell.fill = PatternFill(start_color=clr1, end_color=clr1, fill_type="solid")

        clr2 = "FFA500"
        for rows in ws.iter_rows(min_row=2, max_row=3, min_col=3, max_col=8):
            for cell in rows:
                cell.fill = PatternFill(start_color=clr2, end_color=clr2, fill_type="solid")

        for rows in ws.iter_rows(min_row=2, max_row=3, min_col=15, max_col=20):
            for cell in rows:
                cell.fill = PatternFill(start_color=clr2, end_color=clr2, fill_type="solid")

        for rows in ws.iter_rows(min_row=2, max_row=3, min_col=27, max_col=32):
            for cell in rows:
                cell.fill = PatternFill(start_color=clr2, end_color=clr2, fill_type="solid")

        clr3 = "808080"
        for rows in ws.iter_rows(min_row=2, max_row=3, min_col=9, max_col=14):
            for cell in rows:
                cell.fill = PatternFill(start_color=clr3, end_color=clr3, fill_type="solid")

        for rows in ws.iter_rows(min_row=2, max_row=3, min_col=21, max_col=26):
            for cell in rows:
                cell.fill = PatternFill(start_color=clr3, end_color=clr3, fill_type="solid")

        for rows in ws.iter_rows(min_row=2, max_row=3, min_col=33, max_col=38):
            for cell in rows:
                cell.fill = PatternFill(start_color=clr3, end_color=clr3, fill_type="solid")

        # Specification:
        startValue = r + 2
        ws.merge_cells(start_row=startValue, start_column=1, end_row=startValue + 2, end_column=1)
        ws.cell(row=startValue, column=1).value = "PASS"
        ws.cell(row=startValue, column=1).fill = PatternFill(start_color='90EE90', end_color='90EE90',
                                                             fill_type="solid")
        ws.cell(row=startValue, column=2).value = "Better but within spec."
        ws.cell(row=startValue, column=2).fill = PatternFill(start_color='90EE90', end_color='90EE90',
                                                             fill_type="solid")
        ws.cell(row=startValue + 1, column=2).value = "Worse but within spec."
        ws.cell(row=startValue + 1, column=2).fill = PatternFill(start_color='FFFF00', end_color='FFFF00',
                                                                 fill_type="solid")
        ws.cell(row=startValue + 2, column=2).value = "Beating spec."
        ws.cell(row=startValue + 2, column=2).fill = PatternFill(start_color='ADD8E6', end_color='ADD8E6',
                                                                 fill_type="solid")
        ws.cell(row=startValue + 3, column=1).value = "FAIL"
        ws.cell(row=startValue + 3, column=1).fill = PatternFill(start_color='FF0000', end_color='FF0000',
                                                                 fill_type="solid")
        ws.cell(row=startValue + 3, column=2).value = "Failing spec."
        ws.cell(row=startValue + 3, column=2).fill = PatternFill(start_color='FF0000', end_color='FF0000',
                                                                 fill_type="solid")

        wb.save(excelFile)

    return True

def rerun_complete_batch_for_no_license(sftp_client=None,boardid=0,rev="",comp_name="",files_path="",local_folder="",remote_folder=""):

	inbound_files = sftp_client.listdir(files_path)

	if not os.path.exists(local_folder):
		os.makedirs(local_folder)

	for file_name in inbound_files:
		
		# check for log file
		if file_name.startswith(comp_name) and file_name.endswith(".log"):

			remote_log_file_path = os.path.join(files_path,file_name)
			local_log_file_path = os.path.join(local_folder,file_name)
			
			sftp_client.get(remote_log_file_path,local_log_file_path)

			time.sleep(2)

			if os.path.isfile(local_log_file_path):
				log_file = ""
				try:
					with open(local_log_file_path,"r") as f:
						log_file = f.readlines()
						f.close()
				except Exception as inst:
					if is_logging:
						logging.exception('')
					print("Error reading log file: ",inst)

				for line in log_file:

					if no_license_available in line:

						print("Error in ExtractIM: ",no_license_available)

						is_error = False

						# removing log file on remote rds machine
						sftp_client.remove(os.path.join(files_path,file_name))

						# removing log file from local smb location
						os.remove(local_log_file_path)

						print("Re-running batch file")

						# re-execute main bacth file
						cmd_to_trigger_run = copy.deepcopy(os.path.join(remote_folder,"litepi_jobs.bat"))
						print("executing cmd: ",cmd_to_trigger_run)

						if is_lsf_machine:
							exec_command_through_ssh(server=windows_lsf_rds,cmd=cmd_to_trigger_run,need_response=False,close_ssh=False)
						else:
							exec_command_through_ssh(server="windows",cmd=cmd_to_trigger_run,need_response=False,close_ssh=False)

						return True


	return False

def rerun_compare(sftp_client=None,boardid=0,rev="",comp_name=""):

	# write and push into server for trigger the run
	json_file_base = os.path.join(cloud_base_url,"litepi","temp","Board_ID_"+str(boardid),rev,comp_name)
	batch_file_path = os.path.join(json_file_base,"litepi_rerun_compare_job.bat")

	rerun_compare_base_batch_file_path = os.path.join(cloud_base_url,"litepi","batch_file","litepi_jobs_lsf_amr_drive_rerun_compare.bat")
	
	with open(rerun_compare_base_batch_file_path,"r") as f:
		batch_file_data = f.readlines()

	with open(batch_file_path,"w") as f:

		if is_lsf_machine:

			if is_amr_drive:

				set_cmd = "SET PROJECT="+os.path.join(lsf_path,"Board_ID_"+str(boardid),rev,comp_name,"config_file.json")
				f.write(set_cmd.replace("/","\\"))

				f.write("\n")
				f.write("\n")

		for line in batch_file_data:
			f.write(line)


	remote_file_path = os.path.join(lsf_path,"Board_ID_"+str(boardid),rev,comp_name,"litepi_rerun_compare_job.bat")

	# pushing json file to server
	push_file_to_windows_server(boardid=boardid,rev=rev,comp_name=comp_name,local_file_path=batch_file_path,remote_file_path=remote_file_path,lsf_path=lsf_path)

	cmd_to_trigger_run = copy.deepcopy(remote_file_path)
	print("executing cmd: ",cmd_to_trigger_run)

	if is_lsf_machine:
		exec_command_through_ssh(server=windows_lsf_rds,cmd=cmd_to_trigger_run,need_response=False,close_ssh=False)
	else:
		exec_command_through_ssh(server="windows",cmd=cmd_to_trigger_run,need_response=False,close_ssh=False)

	return True

def get_reports_from_windows_server(sftp_client,source_folder,local_folder):

	if is_smb_drive:
		inbound_files = os.listdir(source_folder)
	else:
		inbound_files = sftp_client.listdir(source_folder)

	for ele in inbound_files:
		if "." in ele:
			try:

				path_from = os.path.join(source_folder,ele)
				path_to = os.path.join(local_folder,ele)

				#print("The destination path of the file is",path_to)
				if not ele.endswith(".spd"):

					if is_smb_drive:
						shutil.copy(path_from, path_to)
					else:
						sftp_client.get(path_from, path_to)
			except Exception as inst:
				if is_logging:
					logging.exception('')
				print(inst)
		else:
			try:
				#print("processing sub folder")
				dest_to = os.path.join(local_folder,ele)
				os.mkdir(dest_to)
				#s_folder = source_folder + '\\' + ele
				s_folder = os.path.join(source_folder,ele)

				if is_smb_drive:
					i_files = os.listdir(s_folder)
				else:
					i_files = sftp_client.listdir(s_folder)
				
				get_reports_from_windows_server(sftp_client=sftp_client,source_folder=s_folder,local_folder=dest_to)
			except Exception as inst:
				if is_logging:
					logging.exception('')
				print(inst)

	return True


@app.route('/download_litepi_reports',methods = ['POST', 'GET'])
def download_litepi_reports(boardid=0,run_no=0):
	print("download_litepi_reports....")
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	if request.method == "POST":
		boardid = request.form.get("boardid")
		run_no = request.form.get("run_no")
		comp_id = request.form.get("comp_id")

	elif request.method == "GET":
		boardid = request.args.get("boardid")
		run_no = request.args.get("run_no")
		comp_id = request.args.get("comp_id")

	if boardid is None:
		boardid = 0

	if run_no is None:
		run_no = 0

	if comp_id is None:
		comp_id = 0

	if str(boardid) != str(0) and str(run_no) == str(0):
		sql="SELECT Files FROM LitePiReportsFileStorage WHERE BoardID = %s ORDER BY RunNo DESC LIMIT 1"
		val=(boardid,)
		d=execute_query(sql,val)
	else:
		sql="SELECT Files FROM LitePiReportsFileStorage WHERE BoardID = %s AND RunNo = %s AND ComponentID = %s"
		val=(boardid,run_no,comp_id)
		d=execute_query(sql,val)
	
	sql = "SELECT LitePiComponentName FROM ComponentType WHERE ComponentID = %s"
	val = (comp_id,)
	comp_rs=execute_query(sql,val)

	if comp_rs != ():
		comp_name = comp_rs[0][0]
	else:
		comp_name = ""

	if(d != ()):
		if comp_id in [0,'0',"",None]:
			filename = "LitePI_Reports_Board_ID_"+str(boardid)+"_Rev"+str(run_no)+".zip"
		else:
			filename = "LitePI_Reports_Board_ID_"+str(boardid)+"_Rev"+str(run_no)+"_"+str(comp_name)+".zip"

		return send_file(BytesIO(d[0][0]),download_name=filename,as_attachment=True)
	else:
		return render('error_custom.html',error='No Files are available to download.',username=username,user_role_name=user_role_name,region_name=region_name)

@app.route("/get_litepi_summary_details",methods = ['POST', 'GET'])
def get_litepi_summary_details():

	wwid = session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	result = {}
	result["rs"] = []

	if request.method == "POST":
		boardid = request.form.get("boardid",type=str)

		#sql = "SELECT DISTINCT a.BoardID,a.RunNo,a.TriggeredOn,IFNULL(b.UserName,''),a.IsRunning,IFNULL(e.FileName,''),a.Status FROM LitePiRunDetails a LEFT JOIN HomeTable b ON a.TriggeredBy = b.WWID LEFT JOIN LitePiRunFileMappings d ON d.BoardID = a.BoardID AND d.RunNo = a.RunNo LEFT JOIN LitePiFileStorage e ON e.FileID = d.NewPiFileID  WHERE a.BoardID = %s ORDER BY a.RunNo DESC"
		sql = "SELECT DISTINCT a.BoardID,a.RunNo,a.TriggeredOn,IFNULL(b.UserName,''),a.IsRunning,IFNULL(e.FileName,''),IFNULL(f.FileName,''),a.Status FROM LitePiRunDetails a LEFT JOIN HomeTable b ON a.TriggeredBy = b.WWID LEFT JOIN LitePiFileStorage e ON e.FileID = a.NewFileID LEFT JOIN LitePiFileStorage f ON f.FileID = a.RefFileID  WHERE a.BoardID = %s ORDER BY a.RunNo DESC"
		val = (boardid,)
		rs = execute_query(sql,val)

		for row in rs:
			temp = []
			temp.append(row[0])
			temp.append(row[1])
			trigger_date = str(get_work_week_fun_with_year(date_value=row[2])) + " - " +str(row[2].strftime('%H:%M:%S'))
			temp.append(trigger_date)
			temp.append(row[3])

			# for status
			if row[7] == "Completed":
				temp.append('<font color="green"><b>Completed</b></font>')
			
			elif row[7] == "Error":
				temp.append('<font color="red"><b>Error</b></font>')
			
			elif row[7] == "Pending":
				temp.append('<font color="#CC338B"><b>Pending</b></font>')
			
			else:
				temp.append('<font color="#D9B611"><b>Running</b></font>')

			# for download option
			if (row[4] != "yes"):
				temp.append(True)
			else:
				temp.append(False)

			temp.append(row[5])	# New File Name
			
			if row[6] == "":	# ref file name
				temp.append("-")
			else:
				temp.append(row[6])

			result["rs"].append(temp)

	return jsonify(result)

@app.route('/download_stackup_details',methods = ['POST', 'GET'])
def download_stackup_details():
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')

	boardid = 0
	file_id = 0

	if request.method == "POST":
		boardid = request.form.get("boardid")
		file_id = request.form.get("file_id")

	sql = "SELECT BoardID,FileID,SequenceNo,StackupName,LayerType,Thickness,Conductivity,Permittivity,LossTangent,UpdatedOn,UpdatedBy FROM LitePiStackupDetails  WHERE BoardID = %s AND FileID = %s ORDER BY SequenceNo ASC"
	val = (boardid,file_id)
	rs = execute_query(sql,val)

	dataList = []

	i=0
	for row in rs:
		temp = []
		i+=1
		temp.append(i)
		temp.append(row[3])
		temp.append(row[4])
		temp.append(float(row[5]))
		temp.append(float(row[6]))
		temp.append(float(row[7]))
		temp.append(float(row[8]))

		dataList.append(temp)

	wb = openpyxl.Workbook()
	sheet = wb.active

	# first row of the excel sheet
	headeColor = '0071C5'
	lightGray = 'F5F5F5'
	white = 'FFFFFF'

	sheet.cell(1, 1, 'S.no')
	sheet.cell(1, 1).alignment = Alignment(horizontal="center", vertical="center")
	sheet.cell(1, 1).fill = PatternFill(start_color=headeColor, end_color=headeColor, fill_type="solid")
	sheet.cell(1, 1).font = Font(color=white)

	sheet.cell(1, 2, 'Name')
	sheet.cell(1, 2).alignment = Alignment(horizontal="center", vertical="center")
	sheet.cell(1, 2).fill = PatternFill(start_color=headeColor, end_color=headeColor, fill_type="solid")
	sheet.cell(1, 2).font = Font(color=white)

	sheet.cell(1, 3, 'Layer Type')
	sheet.cell(1, 3).alignment = Alignment(horizontal="center", vertical="center")
	sheet.cell(1, 3).fill = PatternFill(start_color=headeColor, end_color=headeColor, fill_type="solid")
	sheet.cell(1, 3).font = Font(color=white)

	sheet.cell(1, 4, 'Thickness (µm)')
	sheet.cell(1, 4).alignment = Alignment(horizontal="center", vertical="center")
	sheet.cell(1, 4).fill = PatternFill(start_color=headeColor, end_color=headeColor, fill_type="solid")
	sheet.cell(1, 4).font = Font(color=white)

	sheet.cell(1, 5, 'Conductivity')
	sheet.cell(1, 5).alignment = Alignment(horizontal="center", vertical="center")
	sheet.cell(1, 5).fill = PatternFill(start_color=headeColor, end_color=headeColor, fill_type="solid")
	sheet.cell(1, 5).font = Font(color=white)

	sheet.cell(1, 6, 'Permittivity')
	sheet.cell(1, 6).alignment = Alignment(horizontal="center", vertical="center")
	sheet.cell(1, 6).fill = PatternFill(start_color=headeColor, end_color=headeColor, fill_type="solid")
	sheet.cell(1, 6).font = Font(color=white)

	sheet.cell(1, 7, 'Loss Tangent')
	sheet.cell(1, 7).alignment = Alignment(horizontal="center", vertical="center")
	sheet.cell(1, 7).fill = PatternFill(start_color=headeColor, end_color=headeColor, fill_type="solid")
	sheet.cell(1, 7).font = Font(color=white)

	row = 2
	column = 1
	for data in dataList:
		column = 1
		for val in data:
			sheet.cell(row=row, column=column, value=val)
			column += 1
		row += 1
	maxRow = row

	for rows in sheet.iter_rows(min_row=2, max_row=maxRow, min_col=1, max_col=7):
		for cell in rows:
			if cell.row % 2 == 0:
				cell.fill = PatternFill(start_color=white, end_color=white, fill_type="solid")
			else:
				cell.fill = PatternFill(start_color=lightGray, end_color=lightGray, fill_type="solid")

	# setting the border:
	thin = Border(left=Side(border_style="thin", color="000000"),
				  right=Side(border_style="thin", color="000000"),
				  top=Side(border_style="thin", color="000000"),
				  bottom=Side(border_style="thin", color="000000"))
	for x in range(1, maxRow + 1):
		for y in range(1, 8):
			sheet.cell(row=x, column=y).border = thin

	# Total value calculation:
	sheet.merge_cells(start_row=maxRow, start_column=1, end_row=maxRow, end_column=3)
	sheet.cell(row=maxRow, column=1, value="Total")
	sheet.cell(row=maxRow, column=1).alignment = Alignment(horizontal="right", vertical="center")
	total = sheet.cell(row=maxRow, column=4)

	startRange = 'D2'
	endrange = 'D' + str((maxRow - 1))
	total.value = '=SUM(' + startRange + ':' + endrange + ')'

	sheet = adjust_column_width_from_col(sheet, 1, 1, 7)

	# to make non-editable cell in download spreadsheet
	sheet.protection.sheet = True

	for row in sheet.iter_rows(min_row=2,max_row=maxRow-1,min_col=4,max_col=7):
		for c in row:
			c.protection = Protection(locked=False)

		sheet['D'+str(maxRow)].protection = Protection(locked=False)

	#wb.save(os.path.join(cloud_base_url,'DataFile.xlsx'))

	filename = "Stackup_details_Board_ID_"+str(boardid)+"_File_"+str(file_id)+".xlsx"
	
	return send_file(BytesIO(save_virtual_workbook(wb)),attachment_filename=filename,as_attachment=True)

@app.route('/upload_stackup_details',methods = ['POST', 'GET'])
def upload_stackup_details():

	wwid = session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	boardid = 0
	file_id = 0

	result = {}
	result["is_updated"] = False
	result["err_msg"] = ""

	try:
		if request.method == "POST":
			boardid = request.form.get("boardid")
			file_id = request.form.get("file_id")
			
			file = request.files["stackup_file"]
			file_name = secure_filename(file.filename)
			print("file_name: ",file_name)

			#file_path = cloud_base_url + "//temp//cmd.txt"
			folder_name = str(random.randint(1,1000))
			file_path = os.path.join(cloud_base_url,"temp","stackup_files",folder_name,file_name)

			if not os.path.exists(os.path.join(cloud_base_url,"temp","stackup_files",folder_name)):
				os.makedirs(os.path.join(cloud_base_url,"temp","stackup_files",folder_name))

			file.save(file_path)

			book = openpyxl.load_workbook(file)
			workSheet = book.active

			newDataList = []
			for row in workSheet.iter_rows(min_row=2,max_row=workSheet.max_row-1,min_col=1,max_col=7):
				data = []
				for cell in row:
					data.append(str(cell.value).replace(" ",""))
				newDataList.append(data)
			print(newDataList)

			final_stackup_details = []

			for row in newDataList:
				print(row)

				if not check_valid_values_for_stackup(value=row[3]):
					result["err_msg"] = "Invalid Thickness value found in uploaded file."
					return jsonify(result)

				if not check_valid_values_for_stackup(value=row[4]):
					result["err_msg"] = "Invalid Conductivity value found in uploaded file."
					return jsonify(result)

				if not check_valid_values_for_stackup(value=row[5]):
					result["err_msg"] = "Invalid Permittivity value found in uploaded file."
					return jsonify(result)

				if not check_valid_values_for_stackup(value=row[6]):
					result["err_msg"] = "Invalid Loss Tangent value found in uploaded file."
					return jsonify(result)

				temp = []
				temp.append(boardid)
				temp.append(file_id)
				temp.append(row[0])
				temp.append(row[1])
				temp.append(row[2])

				# set default values if input excel sheet doesnt have value, NaN
				if row[3] == 'None':
					temp.append(0)
				else:
					temp.append(row[3])

				if row[4] == 'None':
					temp.append(0)
				else:
					temp.append(row[4])

				if row[5] == 'None':
					temp.append(1)
				else:
					temp.append(row[5])

				if row[6] == 'None':
					temp.append(0)
				else:
					temp.append(row[6])

				temp.append(t)
				temp.append(wwid)

				final_stackup_details.append(temp)

			sql = "SELECT COUNT(*) FROM LitePiStackupDetails WHERE BoardID = %s AND FileID = %s"
			val = (boardid,file_id)
			rs = execute_query(sql,val)

			row_count = 0
			if rs != ():
				row_count = rs[0][0]

			print("len(final_stackup_details): ",len(final_stackup_details))
			print("row_count: ", row_count)

			if row_count == len(final_stackup_details):
				print("row count match")
				sql = "DELETE FROM LitePiStackupDetails WHERE BoardID = %s AND FileID = %s"
				val = (boardid,file_id)
				rs = execute_query(sql,val)

				sql = """INSERT INTO LitePiStackupDetails(BoardID,FileID,SequenceNo,StackupName,LayerType,Thickness,Conductivity,Permittivity,LossTangent,UpdatedOn,UpdatedBy) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)"""
				execute_query_many(sql,final_stackup_details)

				result["is_updated"] = True

			else:
				result["is_updated"] = False
				result["err_msg"] = "Stackup layers are not matching."
				return jsonify(result)

	except Exception as inst:
		print(inst)

	return jsonify(result)

def check_valid_values_for_stackup(value):

	if value in ["",None,'None']:
		return True

	if re.match('[+]?\d+$',value) or re.match('[+]?[0-9]+\.[0-9]+',value):
		return True
	else:
		return False

	return True

@app.route("/design_quality_submit_data",methods = ['POST', 'GET'])
def design_quality_submit_data():

	wwid=session.get('wwid')
	username = session.get('username')

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	data = json.loads(request.form.get("data"))

	result = {}
	
	board_id = data["boardid"]
	comp_id = data["comp_id"]
	area_of_quality_issue = data["area_of_quality_issue"]
	basic_quality_comments = data["basic_quality_comments"]
	quality_not_met_counter = 1

	sql = "SELECT CommentID,IsSubmitted,QualityNotMetCounter FROM BasicDesignQualityCheck WHERE BoardID = %s AND ComponentID = %s ORDER BY CommentID DESC"
	val = (board_id,comp_id)
	rs_temp = execute_query(sql,val)

	print("rs_temp: ",rs_temp)

	valid_email_trigger = False

	if rs_temp != ():

		for row in rs_temp:

			if row[1] in [None,"no",""]:

				print("updating...")

				sql = "UPDATE BasicDesignQualityCheck SET QualityCheck = %s,AreaOfQualityIssue = %s,Comments = %s,QualityNotMetCounter = QualityNotMetCounter + %s,IsSubmitted = %s,UpdatedBy = %s,UpdatedOn = %s WHERE BoardID = %s AND ComponentID = %s AND CommentID = %s"
				val = ("notmet",area_of_quality_issue,basic_quality_comments,1,"yes",wwid,t,board_id,comp_id,row[0])
				execute_query(sql,val)

				valid_email_trigger = True


	if valid_email_trigger:

		sql = "UPDATE ScheduleTableComponent SET ScheduleStatusID = %s WHERE BoardID = %s AND ComponentID = %s"
		val = (2,board_id,comp_id)
		execute_query(sql,val)		

		sql = "SELECT ComponentName FROM ComponentType WHERE ComponentID = %s"
		val = (comp_id,)
		compname_rs = execute_query(sql,val)

		comp_name = ""
		if compname_rs != ():
			comp_name = compname_rs[0][0]


		sql = "SELECT AreaofIssue FROM AreaOfQualityIssue WHERE ID = %s"
		val = (area_of_quality_issue,)
		area_of_issue_rs = execute_query(sql,val)

		area_of_issue = ""
		if area_of_issue_rs != ():
			area_of_issue = area_of_issue_rs[0][0]

		sql = "SELECT IFNULL(MAX(QualityNotMetCounter),1) FROM BasicDesignQualityCheck WHERE BoardID = %s AND ComponentID = %s"
		val = (board_id,comp_id)
		try:
			quality_not_met_counter = execute_query(sql,val)[0][0]
		except:
			quality_not_met_counter = 1

		sql = "SELECT BoardName FROM BoardDetails WHERE BoardID = %s"
		val = (board_id,)
		try:
			boardname = execute_query(sql,val)[0][0]
		except:
			boardname = ''

		email_list = []

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.DesignLeadWWID and b.BoardID AND b.BoardID = %s) "
		val = (board_id,)
		designlist = execute_query(query,val)
		for i in range(len(designlist)):
			eid = designlist[0][1]
			email_list.append(eid)

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.Username=b.DesignManagerWWID and b.BoardID AND b.BoardID = %s) LIMIT 1"
		val = (board_id,)
		designmanager = execute_query(query,val)
		if designmanager != ():
			email_list.append(designmanager[0][1])

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.CADLeadWWID and b.BoardID AND b.BoardID = %s)"
		val = (board_id,)
		cadlist = execute_query(query,val)
		for i in range(len(cadlist)):
			eid = cadlist[0][1]
			email_list.append(eid)

		query = "SELECT a.Username,a.EmailID FROM RequestAccess a, BoardDetails b WHERE (a.WWID=b.PIFLeadWWID and b.BoardID AND b.BoardID = %s)"
		piflist = execute_query(query,val)
		for i in range(len(piflist)):
			eid = piflist[0][1]
			email_list.append(eid)

		# all pif leads
		email_list += get_pif_leads_email_id_by_board_id(boardid=board_id)

		sql = "SELECT EmailID FROM RequestAccess WHERE RoleID= %s AND StateID = 2"
		val=(14,)
		mgmt_list = execute_query(sql,val)				

		for j in mgmt_list:
			email_list.append(j[0])

		sql = "SELECT EmailID FROM RequestAccess WHERE AdminAccess= %s AND StateID = 2"
		val=('yes',)
		admin_list = execute_query(sql,val)				

		for j in admin_list:
			email_list.append(j[0])

		subject = "[ID:"+str(board_id)+"] Basic design quality not met for "+str(comp_name)
		message = '''Hi,<br><br>
		Basic quality check for Design [ID: '''+str(board_id)+'''] - '''+str(boardname)+''' has <font style="color:red"><b>Not Met</b></font>.<br><br>
		<b>Area of quality issue:  </b> '''+str(area_of_issue)+'''<br>
		<b>Comments:  </b> '''+str(basic_quality_comments)+'''<br>
		<b>Quality Not Met counter:  </b> '''+str(quality_not_met_counter)+'''<br>
		<b>Submitted by: </b>'''+str(username)+'''<br><br><br>
		Regards,<br>
		ERAM.<br>
		'''

		email_list = sorted(set(email_list), reverse=True)
		for i in email_list:
			send_mail_html(i,subject,message,email_list)


		# replace the value in ajax
		result['quality_not_met_counter'] = 0
		result['area_of_quality_issue'] = ""
		result['quality_issue_comments_summary'] = ""

		sql = "SELECT a.BoardID,a.ComponentID,a.CommentID,a.QualityCheck,a.AreaOfQualityIssue,a.Comments,a.QualityNotMetCounter,a.IsSubmitted,a.UpdatedBy,a.UpdatedOn,b.AreaofIssue,IFNULL(c.UserName,'') FROM BasicDesignQualityCheck a LEFT JOIN AreaOfQualityIssue b ON a.AreaOfQualityIssue = b.ID LEFT JOIN HomeTable c ON a.UpdatedBy = c.WWID WHERE a.BoardID = %s AND a.ComponentID = %s ORDER BY a.CommentID DESC"
		val = (board_id,comp_id)
		basic_quality_check_rs = execute_query(sql,val)

		for temp_row in basic_quality_check_rs:

			if temp_row[3] == "notmet":
				result['quality_issue_comments_summary'] += '<span style="color:red;">Not met</span> - '+str(temp_row[10])+'<br>'+str(temp_row[5])+'<br>'+'<span style="color:lightGray;">Updated By: '+str(temp_row[11])+'</span><br><br>'
				result['quality_not_met_counter'] += 1

				if result['area_of_quality_issue'] == "":
					result['area_of_quality_issue'] = str(temp_row[10])

	return jsonify(result)

@app.route("/cmd",methods = ['POST', 'GET'])
def cmd():

	wwid = session.get('wwid')
	username = session.get('username')
	user_role_name = session.get('user_role_name')
	region_name = session.get('region_name')
	is_admin = session.get('is_admin')
	error_msg_inactive_user = session.get('inactive_user_msg')

	t =  datetime.datetime.now(tz).strftime('%Y-%m-%d %H:%M:%S')

	# to check for user status - active or inactive
	if session.get('is_inactive_user'):
		return render('error_custom.html',error=error_msg_inactive_user,username=username,user_role_name=user_role_name,region_name=region_name)

	if not is_admin:
		return render('error_custom.html',error="You do not have access.",username=username,user_role_name=user_role_name,region_name=region_name)

	if request.method == "POST":
		file = request.files["cmd_file"]
		file_name = secure_filename(file.filename)

		cmd_region = request.form.get("cmd_exec")
		print("cmd_region: ",cmd_region)

		#file_path = cloud_base_url + "//temp//cmd.txt"
		file_path = os.path.join(cloud_base_url,"temp","cmd.txt")
		file.save(file_path)

		#if not os.path.exists(cloud_base_url+"//temp"):
		#	os.makedirs(cloud_base_url+"//temp")

		if not os.path.exists(os.path.join(cloud_base_url,"temp")):
			os.makedirs(os.path.join(cloud_base_url,"temp"))

		cmd_file = ["ipconfig"]

		if os.path.isfile(file_path):
			try:
				with open(file_path) as f:
					cmd_file = f.readlines()
					f.close()
			except Exception as inst:
				if is_logging:
					logging.exception('')
				print("Error reading file: ",inst)
		else:
			return "Error."

		if cmd_region == "cf":
			for cmd in cmd_file:
				print("Executing command on Cloud Foundry Linux: ",cmd)
				os.system(cmd)
		else:
			for cmd in cmd_file:
				exec_command_through_ssh(server=cmd_region,cmd=cmd,need_response=True,close_ssh=True)

	return render('cmd.html',username=username,user_role_name=user_role_name,region_name=region_name)

'''
# --------------------------------------------------- Start of MSSQL -----------------------------------------------------------
# mssql server connection
if is_localhost:
	import pyodbc

	server = 'sql1137-pg1-in.gar.corp.intel.com,3180'
	database = 'AppSupport'
	username = 'AppSupport_so'
	password = 'FlexApp40'

	mssql_conn = pyodbc.connect('DRIVER={ODBC Driver 17 for SQL Server};SERVER='+server+';DATABASE='+database+';UID='+username+';PWD='+ password)

	def mssql_execute_query_sql(sql):
		print("----------------MSSQL DB:---------------------")
		try:
			global mssql_conn
			cursor = mssql_conn.cursor()
			cursor.execute(sql)
			result = cursor.fetchall()
			print("result  : ",result)
			return result

		except:
			mssql_conn = pyodbc.connect('DRIVER={ODBC Driver 17 for SQL Server};SERVER='+server+';DATABASE='+database+';UID='+username+';PWD='+ password)
			cursor = mssql_conn.cursor()
			cursor.execute(sql)
			result = cursor.fetchall()
			print("result: ",result)
			return result

	sql = "SELECT * FROM [dbo].[TempTest]"
	mssql_result = mssql_execute_query_sql(sql)

	for row in mssql_result:
		print(row)
		sql = "INSERT INTO TempTableMSSQL VALUES (%s,%s,%s,%s,%s)"
		val = (row[0],row[1],row[2],row[3],row[4])
		#execute_query(sql,val)

	url = "https://apis.intel.com/v1/auth/token"

	data = {
		"grant_type": "client_credentials",
		"client_id": "5wI7bStIynAaZyw5efqtIPYwZ9ycJhq0",
		"client_secret": "ilTcHWIrJrtLplIj"
	}

	response = requests.post(url,data=data,verify=False)
	print(response)
	print("access token : ",response.json()["access_token"])
# --------------------------------------------------- End of MSSQL -------------------------------------------------------------
'''

if(__name__ == '__main__'):
	port = int(os.getenv('PORT', '5000'))
	#app.secret_key="\xfd{H\xe5<\x95\xf9\xe3\x96.5\xd1\x01O<!\xd5\xa2\xa0\x9fR\xa1\xa8"
	app.secret_key="\xfd{H\xe5<\x95\xf9\xe3\x96.5\xd1\x01O<!\xd5\xa2\xa0\x9fR\xa9\xa8"

	if is_localhost:
		# to speed up execution time, clear cache limit
		#app.jinja_env.cache = {}
		app.run(debug=True,threaded=True)
		#app.run(host='0.0.0.0',port=port)
		#server = WSGIServer(('0.0.0.0',port), app)
		#server.serve_forever()

	else:
		print("inactive_email_ids before: ", inactive_email_ids)
		set_inactive_emailids()
		print("inactive_email_ids after: ", inactive_email_ids)

		server = WSGIServer(('0.0.0.0',port), app)
		server.serve_forever()