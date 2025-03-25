from flask import*
import pymysql
import pymysql.cursors
import os
# Mpesa Payment Route 
import requests
import datetime
import base64
from requests.auth import HTTPBasicAuth
# cors
from flask_cors import CORS 


app =Flask(__name__)

app.config["UPLOAD_FOLDER"] ='static/images'
CORS(app)

@app.route("/api/signup", methods =["POST"])

def signup():
    username=request.form["username"]
    email=request.form["email"]
    phone=request.form["phone"]
    password=request.form["password"]

    # create db connection
    connection = pymysql.connect(host="Mesh1.mysql.pythonanywhere-services.com", user="Mesh1", password ="modcom1234", database="Mesh1$default")

    # initiliaze db connection using .cursor()
    cursor =connection.cursor()

    # sql query
    sql = "insert into users (username, email, phone, password) values (%s,%s,%s,%s)"
    data =(username, email, phone, password)

    # execute the query
    cursor.execute(sql, data)

    # save the changes
    connection.commit()

    return jsonify({"success" : "Thank you for joining"})

 # sign in option
@app.route("/api/signin", methods=["POST"])
def signin():
    username =request.form["username"]
    password =request.form["password"]
 # create a database connection
    connection =pymysql.connect(host="Mesh1.mysql.pythonanywhere-services.com", user="Mesh1", password="modcom1234", database="Mesh1$default")
    cursor =connection.cursor(pymysql.cursors.DictCursor)
 # how to confirm the user in the database
    sql ="select user_id, username ,email, phone from users where username = %s and password = %s"
    data = (username, password)
    cursor.execute(sql, data)

    if cursor.rowcount == 0:
        return jsonify({"message" : "login failed. Inavlid detils"})
    else:
        user =cursor.fetchone()
        return jsonify({"messge" : "login successfull", "user" :user})

# uploading products
@app.route("/api/addproducts", methods =["POST"])
def add_products():
    product_name =request.form["product_name"]
    product_desc =request.form["product_desc"]
    product_cost =request.form["product_cost"]
    
 # retrive image from form-use.file since image is a file
    photo =request.files["product_photo"]
    # get the name of the photo 
    photo_name =photo.filename
    # save images to images folder
    # 1.specify the full path
    photo_path =os.path.join(app.config["UPLOAD_FOLDER"], photo_name)
    # 2.save the image to specified location
    photo.save(photo_path)

    # proceed to save data to the db
    connection =pymysql.connect(host="Mesh1.mysql.pythonanywhere-services.com", user="Mesh1", password="modcom1234", database="Mesh1$default")
    cursor =connection.cursor()
    sql="insert into products (product_name, product_desc, product_cost, product_photo) values (%s, %s, %s, %s)"
    data =(product_name, product_desc, product_cost, photo)
    cursor.execute(sql, data)

    # save the changes
    connection.commit()

    return jsonify({"success" : "product saved successfully"})

# retriving data from the db
@app.route("/api/getproducts")
def get_products():
    connection =pymysql.connect(host="Mesh1.mysql.pythonanywhere-services.com", user="Mesh1", password="modcom1234", database="Mesh1$default")
    cursor =connection.cursor(pymysql.cursors.DictCursor)
    sql ="select * from products"
    cursor.execute(sql)
    products =cursor.fetchall()
    return jsonify(products)


@app.route('/api/mpesa_payment', methods=['POST'])
def mpesa_payment():
    if request.method == 'POST':
        # Extract POST Values sent
        amount = request.form['amount']
        phone = request.form['phone']

        # Provide consumer_key and consumer_secret provided by safaricom
        consumer_key = "GTWADFxIpUfDoNikNGqq1C3023evM6UH"
        consumer_secret = "amFbAoUByPV2rM5A"

        # Authenticate Yourself using above credentials to Safaricom Services, and Bearer Token this is used by safaricom for security identification purposes - Your are given Access
        api_URL = "https://sandbox.safaricom.co.ke/oauth/v1/generate?grant_type=client_credentials"  # AUTH URL
        # Provide your consumer_key and consumer_secret 
        response = requests.get(api_URL, auth=HTTPBasicAuth(consumer_key, consumer_secret))
        # Get response as Dictionary
        data = response.json()
        # Retrieve the Provide Token
        # Token allows you to proceed with the transaction
        access_token = "Bearer" + ' ' + data['access_token']

        #  GETTING THE PASSWORD
        timestamp = datetime.datetime.today().strftime('%Y%m%d%H%M%S')  # Current Time
        passkey = 'bfb279f9aa9bdbcf158e97dd71a467cd2e0c893059b10f78e6b72ada1ed2c919'  # Passkey(Safaricom Provided)
        business_short_code = "174379"  # Test Paybile (Safaricom Provided)
        # Combine above 3 Strings to get data variable
        data = business_short_code + passkey + timestamp
        # Encode to Base64
        encoded = base64.b64encode(data.encode())
        password = encoded.decode()

        # BODY OR PAYLOAD
        payload = {
            "BusinessShortCode": "174379",
            "Password":password,
            "Timestamp": timestamp,
            "TransactionType": "CustomerPayBillOnline",
            "Amount": "1",  # use 1 when testing
            "PartyA": phone,  # change to your number
            "PartyB": "174379",
            "PhoneNumber": phone,
            "CallBackURL": "https://coding.co.ke/api/confirm.php",
            "AccountReference": "SokoGarden Online",
            "TransactionDesc": "Payments for Products"
        }

        # POPULAING THE HTTP HEADER, PROVIDE THE TOKEN ISSUED EARLIER
        headers = {
            "Authorization": access_token,
            "Content-Type": "application/json"
        }

        # Specify STK Push  Trigger URL
        url = "https://sandbox.safaricom.co.ke/mpesa/stkpush/v1/processrequest"  
        # Create a POST Request to above url, providing headers, payload 
        # Below triggers an STK Push to the phone number indicated in the payload and the amount.
        response = requests.post(url, json=payload, headers=headers)
        print(response.text) # 
        # Give a Response
        return jsonify({"message": "An MPESA Prompt has been sent to Your Phone, Please Check & Complete Payment"})




app.run(debug=True)
