# MachineLearning_MDR
# Import ---------------------------------------------------------------------------------------------------------------
# data handling
import numpy as np
import pandas as pd
from pandas import Series, DataFrame
import pickle
import os
from os import listdir
from os.path import isfile, join
from itertools import chain

# modeling
from sklearn.preprocessing import MinMaxScaler, QuantileTransformer
from sklearn.feature_selection import VarianceThreshold
from sklearn.model_selection import train_test_split, StratifiedKFold, KFold, GridSearchCV
from sklearn.linear_model import LogisticRegression
from sklearn.neural_network import MLPClassifier
from sklearn.ensemble import BaggingClassifier, RandomForestClassifier, GradientBoostingClassifier
from lightgbm.sklearn import LGBMClassifier
from sklearn.pipeline import Pipeline

# balancing class
from imblearn.over_sampling import RandomOverSampler, SMOTE
from imblearn.under_sampling import RandomUnderSampler
####RE
from imblearn import under_sampling, over_sampling
from imblearn.over_sampling import SMOTE

# evaluation
from sklearn.metrics import auc, f1_score, recall_score, accuracy_score, log_loss
from sklearn.metrics import roc_curve, precision_recall_curve, confusion_matrix, classification_report
from time import time

# visualization
import matplotlib.pyplot as plt
import seaborn as sns

sns.set(style='ticks')  # http://seaborn.pydata.org/tutorial/color_palettes.html
color = sns.color_palette()

# % matplotlib
# inline


# Preprocessing --------------------------------------------------------------------------------------------------------
class preprocessing:
    def __init__(self, pwd, num_list, mdr_type):
        self.pwd = pwd
        self.num_list = num_list
        self.mdr_type = mdr_type
        self.threshold = None
        self.max_list = None
        self.min_list = None

    # import and preprocess data
    def input_data(self, threshold=None):
        # read file and split it to x and y
        x = pd.read_csv(self.pwd)
        x.fillna('NA', inplace=True)
        y = x[self.mdr_type]
        x.drop([item for item in x if "MDR_TYPE" in item], axis=1, inplace=True)

        # add the columns which have the word 'useday' to the num_list
        temp = [index for index, value in enumerate(x) if "USEDAY" in value]
        temp = x.columns[temp]
        self.num_list.extend(temp)
        '''
        # manage feature types
        temp = [col for col in x.columns if col not in num_list]
        x[num_list[0]] = x[num_list[0]].astype("float64")
        x[num_list[1:]] = x[num_list[1:]].astype("int64")
        x[temp] = x[temp].astype("object")
        '''
        # one-hot-encode categorical columns
        x = pd.get_dummies(x)

        # apply nearZeroVar()
        if threshold is not None:
            x = self.nearZeroVar(x, threshold)
        return x, y

    # delete features whose variance are under threshold
    def nearZeroVar(self, x, threshold):
        selector = VarianceThreshold(threshold=threshold)
        selector.fit_transform(x)
        temp = x.columns[selector.get_support(indices=True)]
        print("There are", str(len(selector.get_support(indices=True))) + " features whose variance is over",
              self.threshold)
        return x.loc[:, temp]

    # normalize numeric columns of train data
    def scale(self, x):
        num_list = self.num_list
        scale_quant = QuantileTransformer()
        x2 = scale_quant.fit_transform(x[num_list])
        pickle.dump(scale_quant, open("./scaler/scale_quant.sav", 'wb'))
        scale_minmax = MinMaxScaler()
        x2 = scale_minmax.fit_transform(x2)
        pickle.dump(scale_minmax, open("./scaler/scale_minmax.sav", 'wb'))
        x2 = pd.DataFrame(x2, columns=num_list, index=x.index)
        x2 = pd.concat([x2, x.drop(num_list, axis=1)], axis=1)
        return x2

    # normalize numeric columns of test data
    def use_scale(self, x):
        scale_quant = pickle.load(open("./scaler/scale_quant.sav", 'rb'))
        x2 = scale_quant.transform(x.loc[:, self.num_list])
        scale_minmax = pickle.load(open("./scaler/scale_minmax.sav", 'rb'))
        x2 = scale_minmax.transform(x2)
        x2 = pd.DataFrame(x2, columns=self.num_list, index=x.index)
        x2 = pd.concat([x2, x.drop(self.num_list, axis=1)], axis=1)
        return x2


# Modeling -------------------------------------------------------------------------------------------------------------
class modeling:
    def __init__(self, x_train, x_test, y_train, y_test, model_name, sample_method, random_num, fold_num):
        self.x_train = x_train
        self.x_test = x_test
        self.y_train = y_train
        self.y_test = y_test
        self.model_name = model_name
        self.sample_method = sample_method
        self.random_num = random_num
        self.fold_num = fold_num

    # work out class imbalance problem
    def balancing(self, x, y, method):
        switcher = {
            "ROS": (lambda x, y: RandomOverSampler(random_state=self.random_num).fit_sample(x, y)),
            "SMOTE": (lambda x, y: SMOTE(random_state=self.random_num).fit_sample(x, y)),
            "RUS": (lambda x, y: RandomUnderSampler(random_state=self.random_num).fit_sample(x, y))
        }
        return switcher[method](x, y)

    # split data using stratified k-fold cross validation
    def strat_kfold(self):
        tr_index = [];
        vl_index = []
        skf = StratifiedKFold(n_splits=self.fold_num, shuffle=True, random_state=self.random_num)
        skf.get_n_splits(self.x_train, self.y_train)
        for tr, vl in skf.split(self.x_train, self.y_train):
            tr_index.append(tr), vl_index.append(vl)
        return tr_index, vl_index

    # divide over-class data to  balance with the other
    def over_class_split(self):
        n = int(self.y_train.value_counts()[0] / self.y_train.value_counts()[1])
        kf = KFold(n_splits=n, shuffle=True, random_state=random_num)
        temp = self.y_train[self.y_train == 0]
        chunck_index = [item for _, item in kf.split(temp)]
        return chunck_index

    # select model
    def select_model(self, name):
        switcher = {
            "logit": (lambda: LogisticRegression()),
            "nn": (lambda: MLPClassifier(solver='adam', hidden_layer_sizes=(300, 300, 300),
                                         random_state=self.random_num)),
            "bag": (lambda: BaggingClassifier(n_estimators=500, random_state=self.random_num)),
            "rf": (lambda: RandomForestClassifier(n_estimators=500, max_depth=50,
                                                  random_state=self.random_num, n_jobs=-1)),
            "gbm": (lambda: GradientBoostingClassifier()),
            "lgb": (lambda: LGBMClassifier(boosting_type='gbdt'))
        }
        return switcher[name]()

    # make model
    def make_model(self, x, y, name, method, chunck=False):
        print(name, method, "is fitting ...")
        tic = time()
        model = self.select_model(name)
        if not chunck:
            x, y = self.balancing(x, y, method)
        model.fit(x, y)
        toc = time()
        print("time : ", toc - tic)
        return model

    # save cv_models using make_model()
    def save_cv_models(self, pwd, tr_index):
        for name in self.model_name:
            for method in self.sample_method:
                for k in range(self.fold_num):
                    x_tr, y_tr = self.x_train.iloc[list(tr_index[k]), :], self.y_train.iloc[list(tr_index[k])]
                    model = self.make_model(x_tr, y_tr, name, method)
                    filename = pwd + name + "_" + method + "_" + str(k) + '.sav'
                    pickle.dump(model, open(filename, 'wb'))
        return print("Saving cv models is done.")

    # save chunk_models using make_model()
    def save_chunck_models(self, pwd, chunck_index):
        for name in self.model_name:
            for k in range(len(chunck_index)):
                x_tr, y_tr = self.x_train.iloc[chunck_index[k], :], self.y_train.iloc[chunck_index[k]]
                model = self.make_model(x_tr, y_tr, name, method="", chunck=True)
                filename = pwd + name + "_" + str(k) + '.sav'
                pickle.dump(model, open(filename, 'wb'))
        return print("Saving chunck models is done.")

    # save normal_models using make_model()
    def save_normal_models(self, pwd):
        for name in self.model_name:
            for method in self.sample_method:
                model = self.make_model(self.x_train, self.y_train, name, method)
                filename = pwd + name + "_" + method + '.sav'
                pickle.dump(model, open(filename, 'wb'))
        return print("Saving normal models is done.")

    # get the outputs of model
    def use_model(self, model, x, y):
        y_pred_prob = model.predict_proba(x)[:, 1]
        roc = roc_curve(y, y_pred_prob)
        pr = precision_recall_curve(y, y_pred_prob)
        try:
            f_imp = model.feature_importances_
        except AttributeError:
            f_imp = None
        return y_pred_prob, roc, pr, f_imp

    # load cv_models for validation / test data using use_model()
    def load_cv_models(self, pwd, vl=False, vl_index=None):
        files = [f for f in listdir(pwd) if isfile(join(pwd, f))]
        files = [[item for item in files if name + "_" + method in item] for name in self.model_name for method in
                 self.sample_method]
        y_pred_prob = [];
        roc_thres = [];
        pr_thres = [];
        f_list = [];
        tm = []
        for k in range(len(files)):
            tic = time()
            ypp_k = [];
            roc_k = [];
            pr_k = [];
            f_imp_k = []
            for filename in files[k]:
                print(filename, "is loading ...")
                filename = pwd + filename
                model = pickle.load(open(filename, 'rb'))
                i = int(filename[-5])
                if vl:
                    x_vl, y_vl = self.x_train.iloc[list(vl_index[i]), :], self.y_train.iloc[list(vl_index[i])]
                    ypp, roc, pr, f_imp = self.use_model(model, x_vl, y_vl)
                else:
                    ypp, roc, pr, f_imp = self.use_model(model, self.x_test, self.y_test)
                ypp_k.append(ypp);
                roc_k.append(roc);
                pr_k.append(pr)
                if f_imp is not None: f_imp_k.append(f_imp)
            if len(f_imp_k) is not 0: f_list.append(f_imp_k)
            toc = time()
            y_pred_prob.append(ypp_k);
            roc_thres.append(roc_k);
            pr_thres.append(pr_k);
            tm.append(round(toc - tic, 3))
        print("Loading cv models is done.")
        return y_pred_prob, roc_thres, pr_thres, f_list, tm

    # load chunk_models using use_model()
    def load_chunck_models(self, pwd):
        files = [f for f in listdir(pwd) if isfile(join(pwd, f))]
        files = [[item for item in files if name in item] for name in self.model_name]
        y_pred_prob = [];
        roc_thres = [];
        pr_thres = [];
        f_list = [];
        tm = []
        for k in range(len(files)):
            tic = time()
            ypp_k = [];
            roc_k = [];
            pr_k = [];
            f_imp_k = []
            for filename in files[k]:
                print(filename, "is loading ...")
                filename = pwd + filename
                model = pickle.load(open(filename, 'rb'))
                i = int(filename[-5])
                ypp, roc, pr, f_imp = self.use_model(model, self.x_test, self.y_test)
                ypp_k.append(ypp);
                roc_k.append(roc);
                pr_k.append(pr)
                if f_imp is not None: f_imp_k.append(f_imp)
            if len(f_imp_k) is not 0: f_list.append(f_imp_k)
            toc = time()
            y_pred_prob.append(ypp_k);
            roc_thres.append(roc_k);
            pr_thres.append(pr_k);
            tm.append(round(toc - tic, 3))
        print("Loading chunck models is done.")
        return y_pred_prob, roc_thres, pr_thres, f_list, tm

    # load normal_models using use_model()
    def load_normal_models(self, pwd):
        files = [f for f in listdir(pwd) if isfile(join(pwd, f))]
        y_pred_prob = [];
        roc_thres = [];
        pr_thres = [];
        f_list = [];
        tm = []
        for filename in files:
            tic = time()
            print(filename, "is loading ...")
            filename = pwd + filename
            model = pickle.load(open(filename, 'rb'))
            ypp, roc, pr, f_imp = self.use_model(model, self.x_test, self.y_test)
            if f_imp is not None: f_list.append(f_imp)
            toc = time()
            y_pred_prob.append(ypp);
            roc_thres.append(roc);
            pr_thres.append(pr)
            tm.append(round(toc - tic, 3))
        print("Loading normal models is done.")
        return y_pred_prob, roc_thres, pr_thres, f_list, tm

    def stack_model(self, stack_features, cv_vl_df, cv_te_df, vl_index, mean=True, weight=None):
        tic = time()
        x2_train = cv_vl_df.loc[:, stack_features]
        x2_test = cv_te_df.loc[:, stack_features]
        y2_train = self.y_train.iloc[list(chain(*vl_index))]
        temp = [x2_test[col] for col in x2_test]
        if mean:
            y_stack = [np.mean(row) for row in zip(*temp)]
        else:
            y_stack = [np.inner(vec, weight) / sum(weight) for vec in zip(*temp)]
        y_stack = {"y_stack": y_stack}
        y_stack = DataFrame(y_stack, index=x2_test.index)
        toc = time()
        print("stacking time :", toc - tic)
        return y_stack

    def param_set(self, name):
        switcher = {
            "logit": {'C': [0.01, 0.1, 1, 10, 100]},
            "nn": {"hidden_layer_sizes": [(50, 50), (100, 100), (200, 200, 200)]},
            "bag": {"n_estimator": [200, 500, 1000],
                    "max_depth": [3, 10, 50]},
            "rf": {"n_estimator": [200, 500, 1000],
                   "max_depth": [3, 10, 50]},
            "gbm": {"n_estimator": [50, 100, 250, 500, 1000],
                    "learning_rate": [0.01, 0.1],
                    "max_depth": [3, 10, 50],
                    "subsample": [1.0, 0.7, 0.5, 0.3]},
            "lgb": {"boosting_type": ["gbdt", "dart", "goss"],
                    "num_iterations": [100, 250, 500],
                    "learning_rate": [0.01, 0.1]}
        }
        return switcher[name]


# Merging --------------------------------------------------------------------------------------------------------------
class merging:
    def __init__(self, model_name, sample_method, feature_list):
        self.model_name = model_name
        self.sample_method = sample_method
        self.feature_list = feature_list

    # merge y_pred_prob of cv_models for validation data
    def cv_vl_pred(self, pwd, index, y_pred_prob):
        x2 = []
        files = [f[:-4] for f in listdir(pwd) if isfile(join(pwd, f))]
        files = [[item for item in files if name + "_" + method in item] for name in self.model_name for method in
                 self.sample_method]
        for k in range(len(files)):
            x2.append({files[k][0][:-2]: list(chain(*y_pred_prob[k]))})
            x2[k] = DataFrame(x2[k], index=index)
        x2 = pd.concat([df for df in x2], axis=1)
        return x2

    # merge y_pred_prob  of cv_models for test data
    def cv_te_pred(self, pwd, index, y_pred_prob):
        x2 = []
        files = [f[:-4] for f in listdir(pwd) if isfile(join(pwd, f))]
        files = [[item for item in files if name + "_" + method in item] for name in self.model_name for method in
                 self.sample_method]
        for k in range(len(files)):
            temp = [np.mean(item) for item in zip(*y_pred_prob[k])]
            x2.append({files[k][0][:-2]: temp})
            x2[k] = DataFrame(x2[k], index=index)
        x2 = pd.concat([df for df in x2], axis=1)
        return x2

    # merge y_pred_prob of chunck_models for test data
    def chunck_pred(self, pwd, index, y_pred_prob):
        x2 = []
        files = [f[:-4] for f in listdir(pwd) if isfile(join(pwd, f))]
        files = [[item for item in files if name in item] for name in self.model_name]
        for k in range(len(files)):
            temp = [np.mean(item) for item in zip(*y_pred_prob[k])]
            x2.append({files[k][0][:-2]: temp})
            x2[k] = DataFrame(x2[k], index=index)
        x2 = pd.concat([df for df in x2], axis=1)
        return x2

    # merge y_pred_prob of normal_models for test data
    def normal_pred(self, pwd, index, y_pred_prob):
        x2 = []
        files = [f[:-4] for f in listdir(pwd) if isfile(join(pwd, f))]
        x2 = {key: value for key, value in zip(files, y_pred_prob)}
        x2 = DataFrame(x2, index=index)
        return x2

    # merge feature importances of cv_models for test data
    def cv_te_features(self, pwd, f_list):
        files = [f[:-4] for f in listdir(pwd) if isfile(join(pwd, f))]
        files = [[item for item in files if name + "_" + method in item] for name in ["gbm", "lgb", "rf"] for method in
                 sample_method]
        files = list(chain(*files))
        f_list = list(chain(*f_list))
        df = {key: value for key, value in zip(files, f_list)}
        df = DataFrame(df, index=self.feature_list)
        return df

    # merge feature importances of normal_models for test data
    def normal_features(self, pwd, f_list):
        files = [f[:-4] for f in listdir(pwd) if isfile(join(pwd, f))]
        files = [col for col in files if any(item in col for item in ["gbm", "lgb", "rf"])]
        df = {key: value for key, value in zip(files, f_list)}
        df = DataFrame(df, index=self.feature_list)
        return df


# Measuring ------------------------------------------------------------------------------------------------------------
class measuring:
    def __init__(self, y_te):
        self.y_te = y_te

    def get_measures(self, y_pred_prob):
        y_pred = np.where(y_pred_prob >= 0.5, 1, 0)
        y_te = self.y_te
        accuracy = round(accuracy_score(y_te, y_pred), 2)
        recall = round(recall_score(y_te, y_pred), 2)
        f1 = round(f1_score(y_te, y_pred), 2)
        logloss = round(log_loss(y_te, y_pred_prob), 2)
        return accuracy, recall, f1, logloss

    def measure_table(self, name, df, loading_time):
        accuracy = [];
        recall = [];
        f1 = [];
        logloss = []
        for col in df:
            a, b, c, d = self.get_measures(df[col])
            accuracy.append(a), recall.append(b), f1.append(c), logloss.append(d)
        temp = {"accuracy": accuracy, "recall": recall, "f1": f1, "logloss": logloss, "loading_time": loading_time}
        df = DataFrame(temp, index=list(df))
        df.to_csv(pwd + name + "_measure_table.csv", index_label="model")
        return df

    def print_model(self, y_pred_prob, table=False):
        y_pred = np.where(y_pred_prob >= 0.5, 1, 0)
        y_te = self.y_te
        print("accuracy : ", round(accuracy_score(y_te, y_pred), 2))
        print("recall   : ", round(recall_score(y_te, y_pred), 2))
        print("f1-score : ", round(f1_score(y_te, y_pred), 2))
        print("log-loss : ", round(log_loss(y_te, y_pred_prob), 2))
        if table:
            print("confusion matrix")
            print(confusion_matrix(y_te, y_pred, labels=[0, 1]))
            print(classification_report(y_te, y_pred))
        return None

    def print_base_model(self, name, y_pred_prob, roc, pr, f_imp=None, features_list=None):
        fpr, tpr, roc_thresholds = roc
        precision, recall, pr_thresholds = pr
        y_pred = np.where(y_pred_prob >= 0.5, 1, 0)
        self.average_measures(fpr, tpr, y_pred)
        self.roc_plot(name, fpr, tpr)
        self.pr_threshold(precision, recall, pr_thresholds)
        if f_imp is not None:
            self.variable_importance(f_imp, features_list)
        return None

    def average_measures(self, fpr, tpr, y_pred):
        y_te = self.y_te
        print("accuracy : ", round(accuracy_score(y_te, y_pred), 2))
        print("auc      : ", round(auc(fpr, tpr), 2))
        print("recall   : ", round(recall_score(y_te, y_pred), 2))
        print("f1-score : ", round(f1_score(y_te, y_pred), 2))
        print("log-loss : ", round(log_loss(y_te, self.y_pred_prob), 2))
        return None

    def roc_plot(self, name, fpr, tpr):
        plt.plot([0, 1], [0, 1], 'k--')
        plt.plot(fpr, tpr, label=name)
        plt.xlabel('False positive rate')
        plt.ylabel('True positive rate')
        plt.title('ROC curve')
        plt.legend(loc='best')
        plt.show()
        return None

    def pr_threshold(self, precisions, recalls, thresholds):
        plt.figure(figsize=(8, 8))
        plt.title("Precision and Recall Scores as a function of the decision threshold")
        plt.plot(thresholds, precisions[:-1], "b--", label="Precision")
        plt.plot(thresholds, recalls[:-1], "g-", label="Recall")
        plt.ylabel("Score")
        plt.xlabel("Decision Threshold")
        plt.legend(loc='best')
        return None

    def variable_importance(self, f_imp, features_list):
        feature_importance = f_imp
        sorted_idx = np.argsort(feature_importance)
        plt.figure(figsize=(15, 15))
        plt.barh(range(30), feature_importance[sorted_idx][-31:-1], align='center')
        plt.yticks(range(30), features_list[sorted_idx][-31:-1])
        plt.xlabel('Importance')
        plt.title('30 Variable Importances')
        plt.draw()
        plt.show()
        return None


# Main -----------------------------------------------------------------------------------------------------------------
if __name__ == "__main__":
    # whose model do you want to fit?
    mdr = "MDR_TYPE_MRSA"
    #     mdr = "MDR_TYPE_VRE"

    # numeric features except use_days
    num_list = ["AGE_YR", "PRIOR_LOS", "OP_VISIT_1YR", "CURR_LOS"]

    # number of folds for cross validation
    fold_num = 5

    # random seed number
    random_num = 123

    # sampling method to work out class imbalance issue
    sample_method = ["RUS"]

    # kinds of models you want to fit
    model_name = ["logit", "gbm", "lgb", "nn", "bag", "rf"]

    # ditinguish data using date
    date = "20181018"

    # directory
    pwd = "./" + mdr + "_" + date + "/"
    pwd_data = "./MDR_data/" + date + "_MRSA.csv"
    # pwd_data =  "./MDR_data/"+date+"_VRE.csv"
    pwd_cv = pwd + "modelcv/"
    #     pwd_chunck = pwd + "chunck/"
    pwd_normal = pwd + "normal/"

    # create new folder
    if not os.path.exists(pwd):
        os.makedirs(pwd)
        os.makedirs(pwd_data)
        # os.makedirs(pwd_chunck)

    if not os.path.exists(pwd_cv):
        os.makedirs(pwd_cv)

    if not os.path.exists(pwd_normal):
        os.makedirs(pwd_normal)

    if not os.path.exists("./scaler"):
            os.makedirs("./scaler")

    # import data, threshold : nearZeroVar()
    data = preprocessing(pwd_data, num_list, mdr)
    x, y = data.input_data(threshold=None)

    # split the original dataset
    x_train, x_test, y_train, y_test = train_test_split(x, y, test_size=0.3, random_state=random_num, stratify=y)

    # scale x_train
    x_train = data.scale(x_train)

    # scale x_test
    x_test = data.use_scale(x_test)

    # k-fold cross validation
    model = modeling(x_train, x_test, y_train, y_test, model_name, sample_method, random_num, fold_num)
    tr_index, vl_index = model.strat_kfold()

    # chuncking train data
    #     chunck_index = model.over_class_split()


    # save models
    model.save_cv_models(pwd_cv, tr_index)
#     model.save_chunck_models(pwd_chunck, chunck_index)
    model.save_normal_models(pwd_normal)


    # load models
    y_pred_prob_vl, roc_thres_vl, pr_thres_vl, f_list_vl, loading_time_vl = model.load_cv_models(pwd_cv, vl=True, vl_index=vl_index)
    y_pred_prob_cv_te, roc_thres_cv_te, pr_thres_cv_te, f_list_cv_te, loading_time_cv_te = model.load_cv_models(pwd_cv)
#     y_pred_prob_ch_te, roc_thres_ch_te, pr_thres_ch_te, f_list_ch_te, loading_time_ch_te = model.load_chunck_models(pwd_chunck)
    y_pred_prob_te, roc_thres_te, pr_thres_te, f_list_te, loading_time_te = model.load_normal_models(pwd_normal)


    # merge y_pred_prob
    merge = merging(model_name, sample_method, x_train.columns)
    cv_vl_df = merge.cv_vl_pred(pwd_cv, x_train.iloc[list(chain(*vl_index)),:].index, y_pred_prob_vl)
    cv_te_df = merge.cv_te_pred(pwd_cv, x_test.index, y_pred_prob_cv_te)
#     chunck_df = merge.chunck_pred(pwd_chunck, x_test.index, y_pred_prob_ch_te)
    normal_df = merge.normal_pred(pwd_normal, x_test.index, y_pred_prob_te)

    # save y_pred_prob table to file
    cv_vl_df.to_csv(pwd+"cv_vl_df.csv", index_label="index")
    cv_te_df.to_csv(pwd+"cv_te_df.csv", index_label="index")
#     chunck_df.to_csv(pwd+"chunck_df.csv", index_label="index")
    normal_df.to_csv(pwd+"normal_df.csv", index_label="index")


    # make measure table
    measure = measuring(y_test)
    cv_te_measure = measure.measure_table("cv_te", cv_te_df, loading_time_cv_te)
#     chunck_measure = measure.measure_table("chunck", chunck_df, loading_time_ch_te)
    normal_measure = measure.measure_table("normal", normal_df, loading_time_te)


    # merge feature importance
    merge = merging(model_name, sample_method, x_train.columns)
    cv_te_f = merge.cv_te_features(pwd_cv, f_list_cv_te)
    te_f = merge.normal_features(pwd_normal, f_list_te)

    # save feature importance table to file
    cv_te_f.to_csv(pwd+"cv_te_feature_importances.csv", index_label="feature")
    te_f.to_csv(pwd+"te_feature_importances.csv", index_label="feature")


    # load y_pred_prob table from disk
    cv_vl_df = pd.read_csv(pwd+"cv_vl_df.csv").set_index(["index"])
    cv_te_df = pd.read_csv(pwd+"cv_te_df.csv").set_index(["index"])
#     chunck_df = pd.read_csv(pwd+"chunck_df.csv").set_index(["index"])
    normal_df = pd.read_csv(pwd+"normal_df.csv").set_index(["index"])


    # stack using weighted sum
    stack_features = ["lgb_RUS"]  #  ["gbm_RUS", "lgb_RUS", "logit_RUS", "rf_RUS"]
    y_stack = model.stack_model(stack_features, cv_vl_df, cv_te_df, vl_index, mean=True)
    # y_pred_prob, roc, pr, f_imp = model.use_model(y_stack)

    measure = measuring(y_test)
    measure.print_model(y_stack, table=True)
    # measure.print_base_model("Test", y_pred_prob, roc, pr, f_imp, features_list=None)
