# odoo-reportbro
odoo集成reportbro实现单据套打、批量打印、条码及图片打印等
# 服务端：
1、虚拟机Ubuntu16.04.4

2、sftp上传文件状态错误（权限）：sudo chmod 777 reportbro/

3、sudo apt install python-pip

4、$ pip install reportbro-lib

5、$ pip install SQLAlchemy tornado

6、$ python reportbro_server.py

这样测试服务是没啥问题，但是还需要处理中文问题！

显示已安装模块：pydoc modules

安装成功！！！一直出问题原因是因为安装的reportbro-lib版本问题！！！

Postman服务测试文件：

{
"report":{
"docElements": [{"borderColor": "#000000", "borderTop": false, "cs_borderColor": "#000000", "cs_fontSize": 12, "elementType": "text", "x": 210, "cs_borderTop": false, "cs_paddingBottom": 2, "height": 20, "cs_paddingLeft": 2, "paddingTop": 2, "alwaysPrintOnSamePage": true, "paddingBottom": 2, "borderLeft": false, "paddingRight": 2, "cs_paddingRight": 2, "cs_bold": false, "cs_condition": "", "cs_underline": false, "cs_borderLeft": false, "cs_verticalAlignment": "top", "cs_horizontalAlignment": "left", "printIf": "", "pattern": "", "italic": false, "cs_borderRight": false, "cs_borderAll": false, "eval": false, "borderWidth": 1, "cs_paddingTop": 2, "cs_lineSpacing": 1, "spreadsheet_addEmptyRow": false, "paddingLeft": 2, "borderAll": false, "width": 100, "cs_font": "helvetica", "backgroundColor": "", "underline": false, "fontSize": 12, "id": 230, "cs_borderBottom": false, "spreadsheet_column": "", "cs_italic": false, "bold": false, "borderBottom": false, "lineSpacing": 1, "cs_borderWidth": "1", "font": "firefly", "containerId": "0_content", "content": "\u4e2d\u6587\u4e2d\u6587\u4e2d\u6587", "cs_styleId": "", "styleId": "", "cs_textColor": "#000000", "verticalAlignment": "top", "horizontalAlignment": "left", "spreadsheet_hide": false, "cs_backgroundColor": "", "textColor": "#000000", "y": 50, "borderRight": false, "removeEmptyElement": false
}], 
"styles": [], 
"parameters": [], 
"version": 2,
"documentProperties":{"contentHeight": "", "footer": true, "unit": "mm", "pageWidth": "", "marginTop": "20", "headerSize": "60", "patternLocale": "de", "header": true, "footerSize": "60", "footerDisplay": "always", "orientation": "portrait", "marginBottom": "10", "headerDisplay": "always", "marginLeft": "20", "pageHeight": "", "pageFormat": "A4", "marginRight": "20", "patternCurrencySymbol": 1}
},
"outputFormat": "pdf",
"data":{},
"isTestData": false
}

![image](https://github.com/inspurodoo/odoo-reportbro/blob/master/static/description/design.png)
![image](https://github.com/inspurodoo/odoo-reportbro/blob/master/static/description/preview.png)

# 服务端代码

import datetime, decimal, json, os, uuid
import tornado.ioloop
import tornado.web
from tornado.web import HTTPError
from sqlalchemy import create_engine, select
from sqlalchemy import Table, Column, BLOB, Boolean, DateTime, Integer, String, Text, MetaData
from sqlalchemy import func
from reportbro import Report, ReportBroError

SERVER_PORT = 8000
SERVER_PATH = r"/reportbro/report/run"
MAX_CACHE_SIZE = 500 * 1024 * 1024  # keep max. 500 MB of generated pdf files in sqlite db


engine = create_engine('sqlite:///:memory:', echo=False)
db_connection = engine.connect()
metadata = MetaData()
report_request = Table('report_request', metadata,
    Column('id', Integer, primary_key=True),
    Column('key', String(36), nullable=False),
    Column('report_definition', Text, nullable=False),
    Column('data', Text, nullable=False),
    Column('is_test_data', Boolean, nullable=False),
    Column('pdf_file', BLOB),
    Column('pdf_file_size', Integer),
    Column('created_on', DateTime, nullable=False))

metadata.create_all(engine)


# method to handle json encoding of datetime and Decimal
def jsonconverter(val):
    if isinstance(val, datetime.datetime):
        return '{date.year}-{date.month}-{date.day}'.format(date=val)
    if isinstance(val, decimal.Decimal):
        return str(val)


class MainHandler(tornado.web.RequestHandler):
    def initialize(self, db_connection):
        self.db_connection = db_connection
        # if using additional fonts then add to this list
        self.additional_fonts = [dict(value='firefly',filename='/home/fireflysung.ttf')]

    def set_access_headers(self):
        self.set_header('Access-Control-Allow-Origin', '*')
        self.set_header('Access-Control-Allow-Methods', 'GET, PUT, OPTIONS')
        self.set_header('Access-Control-Allow-Headers', 'Origin, X-Requested-With, X-HTTP-Method-Override, Content-Type, Accept, Z-Key')

    def options(self):
        # options request is usually sent by browser for a cross-site request, we only need to set the
        # Access-Control-Allow headers in the response so the browser sends the following get/put request
        self.set_access_headers()

    def put(self):
        self.set_access_headers()
        json_data = json.loads(self.request.body.decode('utf-8'))
        report_definition = json_data.get('report')
        output_format = json_data.get('outputFormat')
        if output_format not in ('pdf', 'xlsx'):
            raise HTTPError(400, reason='outputFormat parameter missing or invalid')
        data = json_data.get('data')
        is_test_data = bool(json_data.get('isTestData'))

        try:
            report = Report(report_definition, data, is_test_data, additional_fonts=self.additional_fonts)
        except Exception as e:
            raise HTTPError(400, reason='failed to initialize report: ' + str(e))

        if report.errors:
            return json.dumps(dict(errors=report.errors))
        try:
            now = datetime.datetime.now()

            # delete old reports (older than 3 minutes) to avoid table getting too big
            self.db_connection.execute(report_request.delete().where(
                    report_request.c.created_on < (now - datetime.timedelta(minutes=3))))

            total_size = self.db_connection.execute(select([func.sum(report_request.c.pdf_file_size)])).scalar()
            if total_size and total_size > MAX_CACHE_SIZE:
                # delete all reports older than 10 seconds to reduce db size for cached pdf files
                self.db_connection.execute(report_request.delete().where(
                        report_request.c.created_on < (now - datetime.timedelta(seconds=10))))

            report_file = report.generate_pdf()

            key = str(uuid.uuid4())
            # add report request into sqlite db, this enables downloading the report by url (the report is identified
            # by the key) without any post parameters. This is needed for pdf and xlsx preview.
            self.db_connection.execute(
                report_request.insert(),
                key=key, report_definition=json.dumps(report_definition),
                data=json.dumps(data, default=jsonconverter), is_test_data=is_test_data,
                pdf_file=report_file, pdf_file_size=len(report_file), created_on=now)

            self.write('key:' + key)
        except ReportBroError:
            return json.dumps(dict(errors=report.errors))

    def get(self):
        self.set_access_headers()
        output_format = self.get_query_argument('outputFormat')
        assert output_format in ('pdf', 'xlsx')
        key = self.get_query_argument('key', '')
        report = None
        report_file = None
        if key and len(key) == 36:
            # the report is identified by a key which was saved
            # in an sqlite table during report preview with a PUT request
            row = self.db_connection.execute(select([report_request]).where(report_request.c.key == key)).fetchone()
            if not row:
                raise HTTPError(400, reason='report not found (preview probably too old), update report preview and try again')
            if output_format == 'pdf' and row['pdf_file']:
                report_file = row['pdf_file']
            else:
                report_definition = json.loads(row['report_definition'])
                data = json.loads(row['data'])
                is_test_data = row['is_test_data']
                report = Report(report_definition, data, is_test_data, additional_fonts=self.additional_fonts)
                if report.errors:
                    raise HTTPError(400, reason='error generating report')
        else:
            json_data = json.loads(self.request.body.decode('utf-8'))
            report_definition = json_data.get('report')
            data = json_data.get('data')
            is_test_data = bool(json_data.get('isTestData'))
            if not isinstance(report_definition, dict) or not isinstance(data, dict):
                raise HTTPError(400, reason='report_definition or data missing')
            report = Report(report_definition, data, is_test_data, additional_fonts=self.additional_fonts)
            if report.errors:
                raise HTTPError(400, reason='error generating report')

        try:
            now = datetime.datetime.now()
            if output_format == 'pdf':
                if report_file is None:
                    report_file = report.generate_pdf()
                self.set_header('Content-Type', 'application/pdf')
                self.set_header('Content-Disposition', 'inline; filename="{filename}"'.format(
                    filename='report-' + str(now) + '.pdf'))
            else:
                report_file = report.generate_xlsx()
                self.set_header('Content-Type', 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet')
                self.set_header('Content-Disposition', 'inline; filename="{filename}"'.format(
                    filename='report-' + str(now) + '.xlsx'))
            self.write(report_file)
        except ReportBroError:
            raise HTTPError(400, reason='error generating report')


def make_app():
    return tornado.web.Application([
        (SERVER_PATH, MainHandler, dict(db_connection=db_connection)),
    ])


if __name__ == "__main__":
    app = make_app()
    app.listen(SERVER_PORT)
    tornado.ioloop.IOLoop.current().start()

