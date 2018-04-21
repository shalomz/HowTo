# Python Script to monitor disk space and send an email in case threshold
reached(gmail as provider)

To send a message we are going to use **smtplib** library to dispatch it to
SMTP server

    
    
    **# First we are building message **
    
    
    **from email.mime.text import MIMEText**  
    **msg = MIMEText("Server is running out of disk space")  
    msg["Subject"] = "Low disk space warning"  
    msg["From"] = "admin@example.com"  
    msg["To"] = "test@gmail.com"  
    msg.as_string()**  
    'Content-Type: text/plain; charset="us-ascii"\nMIME-Version: 1.0\nContent-Transfer-Encoding: 7bit\nSubject: Low disk space warning\nTo: admin@example.com\nFrom: admin@example.com\nTo: test@gmail.com\n\nServer is running out of disk space'
    
    
    **# To send a message we need to connect to SMTP server  
    import smtplib**
    
    
    **server=smtplib.SMTP("smtp.gmail.com", 587)**  
    **server.ehlo()**  
    (250, b'smtp.gmail.com at your service, [54.202.39.68]\nSIZE 35882577\n8BITMIME\nSTARTTLS\nENHANCEDSTATUSCODES\nPIPELINING\nCHUNKING\nSMTPUTF8')  
    **server.starttls()**  
    (220, b'2.0.0 Ready to start TLS')  
    **server.login("gmail_username","gmail_password")**  
    (235, b'2.7.0 Accepted')  
    **server.sendmail("admin@example.com","test@gmail.com",msg.as_string())**  
    {}  
    **server.quit()**  
    (221, b'2.0.0 closing connection o76sm39310782pfi.119 - gsmtp')

In case if you are getting error like this

    
    
    server.login(gmail_user, gmail_pwd)  
        File "/usr/lib/python3.4/smtplib.py", line 639, in login  
    **   raise SMTPAuthenticationError(code, resp)  
       smtplib.SMTPAuthenticationError: (534, b'5.7.14 **    
       <https://accounts.google.com/ContinueSignIn?sarp=1&scc=1&plt=AKgnsbtl1\n5.7.14       Li2yir27TqbRfvc02CzPqZoCqope_OQbulDzFqL-msIfsxObCTQ7TpWnbxIoAaQoPuL9ge\n5.7.14 BUgbiOqhTEPqJfb02d_L6rrdduHSxv26s_Ztg_JYYavkrqgs85IT1xZYwtbWIRE8OIvQKf\n5.7.14 xxtT7ENlZTS0Xyqnc1u4_MOrBVW8pgyNyeEgKKnKNyxce76JrsdnE1JgSQzr3pr47bL-kC\n5.7.14 XifnWXg> Please log in via your web browser and then try again.\n5.7.14 Learn more at\n5.7.14 https://support.google.com/mail/bin/answer.py?answer=78754 fl15sm17237099pdb.92 - gsmtp')

Go to this link and select **Turn On**

    
    
    <https://www.google.com/settings/security/lesssecureapps>

Python Script to monitor disk space usage

    
    
    threshold = 90  
    partition = "/"
    
    
    df = subprocess.Popen(["df","-h"], stdout=subprocess.PIPE)  
     for line in df.stdout:  
     splitline = line.decode().split()  
     if splitline[5] == partition:  
     if int(splitline[4][:-1]) > threshold:

Now combine both of them

    
    
    import subprocess  
    import smtplib  
    from email.mime.text import MIMEText
    
    
    threshold = 40  
    partition = "/"
    
    
    def report_via_email():  
     msg = MIMEText("Server running out of disk space")  
     msg["Subject"] = "Low disk space warning"  
     msg["From"] = "admin@example.com"  
     msg["To"] = "test@gmail.com"  
     with smtplib.SMTP("smtp.gmail.com", 587) as server:  
     server.ehlo()  
     server.starttls()  
     server.login("gmail_user","gmail_password)  
     server.sendmail("admin@example.com","test@gmail.com",msg.as_string())
    
    
    def check_once():  
     df = subprocess.Popen(["df","-h"], stdout=subprocess.PIPE)  
     for line in df.stdout:  
     splitline = line.decode().split()  
     if splitline[5] == partition:  
     if int(splitline[4][:-1]) > threshold:  
     report_via_email()  
    check_once()
