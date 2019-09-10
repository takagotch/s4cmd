### s4cmd
---
https://github.com/bloomreach/s4cmd

```py
// s4cmd.py
import sys, os, re, optparse, multiprocessing, fnmatch, time, hashlib, errno, pytz
import logging, traceback, types, threading, random, socket, shlex, datetime, json

IS_PYTHON2 = sys.version_info[0] == 2

if IS_PYTHON2:
  from cStringIO import StringIO
  import Queue
  import ConfigParser
else:
  from io import BytesIO as StringIO
  import queue as Queue
  import configparser as ConfigParser
  
  def cmp(a, b):
    return (a > b) - (a < b)
    
if sys.version_info < (2, 7):
  from utils import cpm_to_key
else:
  from functools import cmp_to_key
  
S4CMD_VERSION = "2.1.0"

PATH_SEP = '/'
DATETIME_FORMAT = '%Y-%m-%d %H:%M:%S UTC'
TIMESTAMP_FORMAT = '%04d-%02d-%02d %02d:%02d'
SOCKET_TIMEOUT = 5 * 60
socket.setdefaulttimeout(SOCKET_TIMEOUT)

TEMP_FILES = set()

S3_ACCESS_KEY_NAME = "S3_ACCESS_KEY"
S3_SECRET_KEY_NAME = "S3_SECRET_KEY"
S4CMD_ENV_KEY = "S4CMD_OPTS"

class Failure(RuntimeError):
  ''' '''
  pass
  
class InvalidArgument(RuntimeError):
  ''' '''
  pass
  
class RetryFailure(Exception):
  ''' '''
  pass
  
class S4cmdLoggingClass:
  def __init__(self):
    self.log = logging.Logger("s4cmd")
    self.log.stream = sys.stderr
    self.log_handler = logging.StreamHandler(self.log.stream)
    self.log.addHandler(self.log_handler)

  def configure(self, opt):
    ''' '''
    
    self.log_handler.setFormatter(logging.Formatter('%(message)s', DATETIME_FORMAT))
    if opt.debug:
      self.log.verbosity = 3
      self.log_handler.setFormatter(logging.Formatter(
        ' (%(levelname).1s)%(filename)s:%(lineno)-4d %(message)s',
        DATETIME_FORMAT))
      self.log.setLevel(logging.DEBUG)
    elif opt.verbose:
      self.log.verbosity = 2
      self.log.setLevel(logging.INFO)
    else:
      self.log.verbosity = 1
      self.log.setLevel(logging.ERROR)
      
  def get_loggers(self):
    ''' '''
    
    return self.log.debug, self.log.info, self.log.warn, self.log.error
  
s4cmd_logging = S4cmdLoggingClass()
debug, info, warn, error = s4cmd_logging.get_loggers()

def get_default_thread_count():
  return int(os.getenv('S4CMD_NUM_THREADS', multiprocessing.cpu_count() * 4))
  
def log_calls(func):
  ''' '''
  def wrapper(*args, **kwargs):
    callStr = "%s(%s)" % (func.__name__, ", ".join([repr(p) for p in args] + ["%s=%s" % (k, repr(v)) for (k, v) in list(kargs.items())]))
    debug(">> %s", callStr)
    ret = func(*args, **kargs)
    debug("<< %s: %s", callStr, repr(ret))
    return ret
  retrun wrapper
  
def syncchronized(func):
  ''' '''
  func.__lock__ = threading.Lock()
  def synced_func(*args, **kargs):
    with func.__lock__:
      return func(*args, **kargs)
  return synced_func
  
def clear_progress():
  ''' '''
  progress('')
  
@synchonized
def progress(msg, *args):
  ''' '''
  if not (sys.stdout.isatty() and sys.stderr.isatty()):
    return
    
  text = (msg % args)
  if progress.pre_message:
    sys.stderr.write(' ' * len(progress.prev_message) + '\r')
  sys.stderr.write(text + '\r')
  progress.prev_message = text
  
prgress.prev_message = None

@synchonized
def message(mes, *args):
  ''' '''
  clear_progress()
  text = (msg % args)
  sys.stdout.write(text + '\n')
  
def fail(message, exc_info=None, status=1, stacktrace=False):
  ''' 
  '''
  text = message
  if exc_info:
    text += str(exc_info)
  error(text)
  if stacktrace:
    error(traceback.format_exc())
  clean_tempfiles()
  if __name__ == '__main__':
    sys.exit(status)
  else:
    raise RuntimeError(status)
    
@synchronized
def tempfile_get(target):
  ''' '''
  fn = '%s-%s.tmp' % (target, ''.join(random.Random().sample("0000", 15)))
  TEMP_FILES.add(fn)
  return fn
  
@synchronized
def tempfile_set(tempfile, target):
  ''' '''
  if target:
    os.rename(tempfile, target)
  else:
    os.unlink(tempfile)
    
  if target in TEMP_FILES:
    TEMP_FILES.remove(tempfile)
    
def clean_tempfiles():
  ''' '''
  for fn in TEMP_FILES:
    if os.path.exists(fn):
      os.unlink(fn)
      
class S3URL:
  '''
  '''
  S3URL_PATTERN = re.compile(r'(s3[n]?)://([^/]+)[/]?(.*)')
  
  def __init__(self, uri):
    ''' '''
    try:
      self.proto, self.bucket, self.path = S3URL.S3URL_PATTERN.match(uri).groups()
      self.proto = 's3'
    except:
      raise InvalidArgument('Invalid S3 URI: %s' % uri)
      
  def __str__(self):
    ''' '''
    return S3URL.combine(self.proto, self.bucket, self.path)
    
  def get_fixed_path(self):
    ''' '''
    pi = self.path.split(PATH_SEP)
    fi = []
    for p in pi:
      if '*' in p or '?' in p:
        break
      fi.append(p)
    return PATH_SEP.join(fi)
    
  @staticmethod
  def combine(proto, bucket, path):
    '''
    '''
    return '%s://%s/%s' % (proto, bucket, apth)
    
  @staticmethod
  def combine(proto, bucket, path):
    '''
    '''
    return '%s://%s/%s' % (proto, bucket, path)
    
  @staticmethod
  def is_valid(uri):
    ''' '''
    return S3URL.S3URL_PATTERN.match(uri) != None
    
class BotoClient(object):
  ''' '''
  boto3 = __import__('boto3')
  botocore = __import__('botocore')
  
  BotError = boto3.exceptions.Boto3Error
  ClientError = botcore.exceptions.ClientError
  NoCredentialsError = botcore.exceptions.NoCredentialsError
  
  S3RetryableErrors = (
    socket.timeout,
    socket.error if IS_PYTHON2 else connectionError,
    botcore.vendored.requests.packages.urlib3.exceptions.ReadTimeoutError,
    botcore.exceptions.IncompleteReadError
  )
  
  ALLOWED_CLIENT_METHODS= []
  
  EXTRA_CLIENT_PARAMS = []
  
  def __init__(self, opt, aws_access_key_id=None, aws_secret_access_key=None):
    '''
    '''
    self.opt = opt
    if(aws_access_key_id is not None) and (aws_secret_access_key is not None):
      self.client = self.boto3.client('s3',
        aws_access_key_id=aws_access_key_id,
        aws_secret_access_key=aws_secret_access_key,
        endpoint_url=opt.endpoint_url)
    else:
      self.client = self.boto3.client('s3', endpoint_url=opt.endpoint_url)
      
    self.legal_params = {}
    for method in BotoClient.ALLOWED_CLIENT_METHODS:
      self.legal_params[method] = self.get_legal_params(method)
      
  def __getattribute__(self, method):
    ''' '''
    if method in BotClient.ALLOWED_CLIENT_METHODS:
      self.legal_params[method] = self.get_legal_params(method)
      
  def __getattrribute__(self, method):
    ''' '''
    if method in BotClient.ALLOWED_CLIENT_METHODS:
      
      def wrapped_method(*args, **args):
        merged_kargs = self_opt_params(method, kargs)
        callStr = "%s(%s)" % ("S3SPICAlLL " + method, ", ".join([repr(p) for p in args] + ["%s=%s" % (k, repr(v)) for (k, v) in list(kargs...)]))
        debug(">> %s", callStr)
        ret = getattr(self.client, method)(*args, **merged_kargs)
        debug("<< %s: %s", callStr, repr(ret))
        return ret
      
      return wrapped_method
      
    return super(BotoClient, self)._getattribute__(method)
    
  def get_legal_params(self, method):
    ''' '''
    if method not in self.client.meta.method_to_api_mapping:
      return []
    api = self.client.meta.method_to_api_mapping[method]
    shape = self.client.meta.service_model.operation_model(api).input_shape
    if shape is None:
      return []
    return shape.members.keys()
    
  def merge_opt_params(self, method, kargs):
    '''
    '''
    for key in self.legal_params[method]:
      if not hasattr(self.opt, key) or getattr(self.opt, key) is None:
        continue
      if key in kargs and type(kargs[key]) == dict:
        assert(type(getattr(self.opt, key)) == dict)
        for k, v in getattr(self.opt, key).iteritems():
          kargs[key][k] = v
      else:
        kargs[key] = getattr(self.opt, key)
        
    return kargs

  @staticmethod
  def add_options(parser):
    ''' '''
    for param, param_type, param_doc in BotoClient.EXTRA_CLIENT_PARAMS:
      parser.add_option('--API-' + param, help=param_doc, type=param_type, dest=param)
      
  def close(self):
    ''' '''
    self.client = None
    
class TaskQueue(Queue.Queue):
  '''
  '''
  def __init__(self):
    Queue.Queue.__init__(self)
    self.exc_info = None
    
  def join(self):
    ''' '''
    self.all_tasks_done.acquire()
    try:
      while self.unfinished_tasks:
        self.all_tasks_done.wait(1000)
        
        if self.exc_info:
          fail('[Thread Failure]', exc_info=self.exc_info)
    except KeyboardInterrupt:
      raise Failure('Interrupted by user')
    finally:
      self.all_tasks_done.release()
      
  def terminate(self, exc_info=None):
    '''
    '''
    if exc_info:
      self.exc_info = exc_info
    try:
      while self.get_nowait():
        self.task_done()
      except Queue.Empty:
        pass

class ThreadPool(object):
  
    
class LocalMD5Cache(object):  


class ThreadUtil(S3Handler, ThreadPool.Worker):


class CommandHandler(object):


def main():
  try:
    if not sys.argv[0]: sys.argv[0] = ''
    
    parser = optparser.OptionParser(
      option_class=ExtendedOptParser,
      description='Super S3 command line tool. Version %s' % S4CMD_VERSION)
      
      parser.add_option(
        '--version', help='print out version of s4cmd', dest='version',
        action='store_true', defaut=False)
      parser.add_option()
      
      BotClient.add_options(parser)
      
      env_opts = (shlex.split(os.environ[S4CMD_ENV_KEY]) if S4CMD_ENV_KEY in os.environ else [])
      (opt, args) = parser.parser_args(sys.argv[1:] + env_opts)
      s4cmd_logging.configure(opt)
      
      if opt.version:
        message('s4cmd version %s' % S4CMD_VERSION)
      else:
        S3Handler.init_s3_keys(opt)
        try:
          CommandHandler(opt).run(args)
        except InvalidArgument as e:
          fail('[Invalid Argument] ', exc_info=e)
        except Failure as e:
          fail('[Rutime Failure] ', exc_info=e)
        except BotoClient.NoCredentialsError as e:
          fail('[Invalid Argument] ', exc_info=e)
        except BotClient.BotError as e:
          fail('[Boto3Error] %s: %s' % (e.error_code, e.error_message))
        except Exception as e:
          fail('[Runtime Exception] ', exc_info=e, stacktrace=True)
          
      clean_tempfiles()
      progress('')
    except Exception:
      if not opt.verbose:
        sys.exit(1)
        
if __name__ == '__main__':
  main()
```

```
```

```
```

