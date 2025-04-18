<template>
  <div class="container mt-4">
    <h2>
      البيانات المالية للقضية رقم: {{ id }}
      <span v-if="caseInfo && caseInfo.case_number"
        >({{ caseInfo.case_number }})</span
      >
    </h2>
    <hr />

    <!-- زر إضافة اتفاقية (يظهر فقط إذا لا توجد اتفاقية) -->
    <div
      class="mb-3"
      v-if="!loading && !error && financialData && !financialData.agreement"
    >
      <b-button variant="primary" @click="showAddAgreementModal">
        <i class="fas fa-plus"></i> إضافة اتفاقية أتعاب
      </b-button>
    </div>

    <!-- مؤشر التحميل -->
    <div v-if="loading" class="text-center">
      <div class="spinner-border text-primary" role="status">
        <span class="visually-hidden">جاري التحميل...</span>
      </div>
      <p>جاري تحميل البيانات المالية...</p>
    </div>

    <!-- رسالة الخطأ -->
    <div v-if="error" class="alert alert-danger" role="alert">
      حدث خطأ أثناء تحميل البيانات: {{ error }}
      <b-button
        size="sm"
        variant="danger"
        class="float-start"
        @click="fetchFinancialData"
      >
        <i class="fas fa-sync-alt"></i> إعادة المحاولة
      </b-button>
    </div>

    <!-- عرض البيانات المالية -->
    <div v-if="!loading && !error && financialData">
      <!-- *** START: قسم الملخص المالي (تم نقله للأعلى وتعديله) *** -->
      <b-card no-body class="mb-3 shadow-sm">
        <b-card-header header-tag="header" class="p-1" role="tab">
          <b-button
            block
            v-b-toggle.collapse-summary
            variant="light"
            class="text-right"
          >
            <span class="float-left"
              ><i class="fas fa-calculator mr-2"></i>الملخص المالي</span
            >
            <i class="fas fa-chevron-down"></i>
          </b-button>
        </b-card-header>
        <b-collapse
          id="collapse-summary"
          visible
          accordion="financial-accordion"
          role="tabpanel"
        >
          <b-card-body>
            <b-row>
              <b-col md="4">
                <b-card
                  bg-variant="light"
                  text-variant="dark"
                  class="text-center mb-2"
                >
                  <b-card-text>
                    <h5 class="mb-1">إجمالي المستحق (المفوتر)</h5>
                    <!-- استخدام الخاصية المحسوبة الجديدة مع فلتر العملة -->
                    <h4>{{ totalDueBilled | currency }}</h4>
                  </b-card-text>
                </b-card>
              </b-col>
              <b-col md="4">
                <b-card
                  bg-variant="success"
                  text-variant="white"
                  class="text-center mb-2"
                >
                  <b-card-text>
                    <h5 class="mb-1">إجمالي المدفوع</h5>
                    <!-- استخدام paymentsTotal الموجودة -->
                    <h4>{{ paymentsTotal | currency }}</h4>
                  </b-card-text>
                </b-card>
              </b-col>
              <b-col md="4">
                <b-card
                  :bg-variant="currentBalance >= 0 ? 'warning' : 'danger'"
                  :text-variant="currentBalance >= 0 ? 'dark' : 'white'"
                  class="text-center mb-2"
                >
                  <b-card-text>
                    <h5 class="mb-1">الرصيد الحالي</h5>
                    <!-- استخدام الخاصية المحسوبة الجديدة مع فلتر العملة -->
                    <h4>{{ currentBalance | currency }}</h4>
                    <small v-if="currentBalance < 0">(دفعة زائدة)</small>
                    <small v-else-if="currentBalance > 0">(مستحق)</small>
                    <small v-else>(مسدد)</small>
                  </b-card-text>
                </b-card>
              </b-col>
            </b-row>
            <!-- يمكن إضافة المزيد من التفاصيل أو الرسوم البيانية هنا لاحقًا -->
          </b-card-body>
        </b-collapse>
      </b-card>
      <!-- *** END: قسم الملخص المالي *** -->

      <!-- قسم اتفاقية الأتعاب -->
      <b-card no-body class="mb-3">
        <b-card-header header-tag="header" class="p-1" role="tab">
          <b-button block v-b-toggle.collapse-agreement variant="info">
            <i class="fas fa-file-contract me-2"></i> اتفاقية الأتعاب والعقد
          </b-button>
        </b-card-header>
        <b-collapse
          id="collapse-agreement"
          accordion="financial-accordion"
          role="tabpanel"
          :visible="financialData && financialData.agreement"
        >
          <b-card-body>
            <!-- Details and Contract section -->
            <div v-if="financialData.agreement">
              <!-- Agreement details display -->
              <p>
                <strong>نوع الاتفاق:</strong>
                {{
                  getAgreementTypeText(financialData.agreement.agreement_type)
                }}
              </p>
              <p v-if="financialData.agreement.total_agreed_amount !== null">
                <strong>المبلغ الإجمالي / المقدم:</strong>
                {{ financialData.agreement.total_agreed_amount | currency }}
              </p>
              <p v-if="financialData.agreement.percentage_rate !== null">
                <strong>النسبة المئوية:</strong>
                {{ financialData.agreement.percentage_rate }} %
              </p>
              <p v-if="financialData.agreement.percentage_basis">
                <strong>أساس احتساب النسبة:</strong>
                {{ financialData.agreement.percentage_basis }}
              </p>
              <p v-if="financialData.agreement.agreement_date">
                <strong>تاريخ الاتفاقية:</strong>
                {{ formatDate(financialData.agreement.agreement_date) }}
              </p>
              <p v-if="financialData.agreement.notes">
                <strong>ملاحظات:</strong> {{ financialData.agreement.notes }}
              </p>

              <!-- Contract document section -->
              <div v-if="financialData.contract_document" class="mt-3">
                <h5>مستند العقد المرفق:</h5>
                <p>
                  <i class="fas fa-file-pdf me-1 text-danger"></i>
                  {{ financialData.contract_document.file_name_original }}
                  <b-link
                    :href="`/api/documents/download_secure.php?doc_id=${financialData.contract_document.document_id}`"
                    target="_blank"
                    class="btn btn-sm btn-outline-primary ms-2"
                    v-b-tooltip.hover
                    title="تحميل العقد"
                  >
                    <i class="fas fa-download"></i> تحميل العقد
                  </b-link>
                  <b-button
                    size="sm"
                    variant="outline-danger"
                    class="ms-2"
                    @click="confirmDeleteContract"
                    v-b-tooltip.hover
                    title="حذف ملف العقد المرفق"
                  >
                    <i class="fas fa-trash-alt"></i> حذف العقد
                  </b-button>
                </p>
              </div>
              <div v-else class="mt-3">
                <p class="text-muted">
                  لم يتم ربط مستند عقد لهذه الاتفاقية بعد.
                </p>
                <b-button
                  size="sm"
                  variant="success"
                  @click="showLinkContractModal"
                  title="رفع وربط ملف العقد PDF بهذه الاتفاقية"
                >
                  <i class="fas fa-upload"></i> رفع وربط ملف العقد
                </b-button>
              </div>

              <!-- Agreement action buttons -->
              <div class="mt-3">
                <b-button
                  size="sm"
                  variant="warning"
                  @click="showEditAgreementModal"
                  v-b-tooltip.hover
                  title="تعديل تفاصيل هذه الاتفاقية"
                >
                  <i class="fas fa-edit"></i> تعديل تفاصيل الاتفاقية
                </b-button>
                <!-- *** START: Agreement Delete Button Modifications *** -->
                <b-button
                  size="sm"
                  variant="danger"
                  class="ms-2"
                  @click="confirmDeleteAgreement"
                  :disabled="isDeletingAgreement"
                  v-b-tooltip.hover
                  title="حذف اتفاقية الأتعاب بالكامل (إذا لم تكن مرتبطة بأقساط)"
                >
                  <b-spinner small v-if="isDeletingAgreement"></b-spinner>
                  <i class="fas fa-times" v-else></i>
                  <span v-if="!isDeletingAgreement" class="ms-1">
                    حذف الاتفاقية
                  </span>
                  <span v-else class="ms-1"> جاري الحذف... </span>
                </b-button>
                <!-- *** END: Agreement Delete Button Modifications *** -->
              </div>
            </div>
            <div v-else>
              <p class="text-muted">
                لم يتم تسجيل اتفاقية أتعاب لهذه القضية بعد.
              </p>
            </div>
          </b-card-body>
        </b-collapse>
      </b-card>
      <!-- === نهاية قسم اتفاقية الأتعاب === -->

      <!-- قسم الأقساط المجدولة -->
      <b-card no-body class="mb-3">
        <b-card-header header-tag="header" class="p-1" role="tab">
          <b-button
            block
            v-b-toggle.collapse-installments
            variant="light"
            class="text-right"
          >
            <span class="float-left">
              <i class="fas fa-tasks mr-2 text-primary"></i>
              الأقساط المجدولة ({{
                financialData.installments
                  ? financialData.installments.length
                  : 0
              }})
            </span>
            <i class="fas fa-chevron-down"></i>
          </b-button>
        </b-card-header>
        <b-collapse
          id="collapse-installments"
          accordion="financial-accordion"
          role="tabpanel"
          :visible="financialData && financialData.agreement"
        >
          <b-card-body>
            <div v-if="financialData.agreement">
              <b-button
                size="sm"
                variant="primary"
                class="mb-3"
                @click="showAddInstallmentModal"
              >
                <i class="fas fa-plus"></i> إضافة قسط مجدول جديد
              </b-button>

              <!-- === جدول عرض الأقساط === -->
              <b-table
                v-if="
                  financialData.installments &&
                  financialData.installments.length > 0
                "
                :items="financialData.installments"
                :fields="installmentTableFields"
                striped
                hover
                responsive
                small
                bordered
                show-empty
                empty-text="لا توجد أقساط مجدولة لعرضها."
                class="mt-3"
                thead-class="table-light"
                tbody-tr-class="text-right"
              >
                <template #cell(amount_due)="data">
                  {{ data.value | currency }}
                </template>

                <template #cell(due_trigger_type)="data">
                  {{ getTriggerTypeText(data.value) }}
                </template>

                <template #cell(due_details)="data">
                  <span v-if="data.item.due_trigger_type === 'SPECIFIC_DATE'">{{
                    formatDate(data.item.due_date) || "-"
                  }}</span>
                  <span
                    v-else-if="
                      data.item.due_trigger_type === 'CUSTOM_EVENT_DESCRIPTION'
                    "
                    >{{ data.item.due_event_description || "-" }}</span
                  >
                  <span v-else>-</span>
                </template>

                <template #cell(status)="{ item }">
                  <b-dropdown
                    :id="`status-dropdown-${item.installment_id}`"
                    size="sm"
                    :variant="getStatusVariant(item.status)"
                    :text="getStatusText(item.status)"
                    :disabled="
                      item.status === 'paid' ||
                      isChangingStatus[item.installment_id]
                    "
                    class="status-dropdown"
                    v-b-tooltip.hover
                    :title="
                      item.status === 'paid'
                        ? 'الحالة مدفوع لا يمكن تغييرها يدويًا'
                        : 'تغيير حالة القسط'
                    "
                    menu-class="dropdown-menu-end"
                    toggle-class="d-flex align-items-center"
                  >
                    <template #button-content>
                      <b-spinner
                        v-if="isChangingStatus[item.installment_id]"
                        small
                        class="me-1"
                      ></b-spinner>
                      <span v-else>{{ getStatusText(item.status) }}</span>
                    </template>
                    <b-dropdown-item
                      v-for="newStatus in allowedManualStatuses"
                      :key="newStatus"
                      @click="
                        changeInstallmentStatus(item.installment_id, newStatus)
                      "
                      :disabled="item.status === newStatus"
                    >
                      تغيير إلى: {{ getStatusText(newStatus) }}
                    </b-dropdown-item>
                  </b-dropdown>
                </template>

                <template #cell(notes)="data">
                  <span :title="data.value">{{
                    truncateText(data.value, 50)
                  }}</span>
                </template>

                <template #cell(actions)="{ item }">
                  <b-button
                    size="sm"
                    variant="outline-warning"
                    class="me-1"
                    @click="showEditInstallmentModal(item)"
                    v-b-tooltip.hover
                    title="تعديل القسط"
                    ><i class="fas fa-edit"></i
                  ></b-button>
                  <b-button
                    size="sm"
                    variant="outline-danger"
                    @click="confirmDeleteInstallment(item)"
                    :disabled="isDeletingInstallment[item.installment_id]"
                    v-b-tooltip.hover
                    title="حذف هذا القسط"
                  >
                    <b-spinner
                      small
                      v-if="isDeletingInstallment[item.installment_id]"
                    ></b-spinner>
                    <i class="fas fa-trash-alt" v-else></i>
                  </b-button>
                </template>
              </b-table>
              <p
                v-else-if="
                  !loading &&
                  financialData.installments &&
                  financialData.installments.length === 0
                "
                class="text-muted"
              >
                لا توجد أقساط مجدولة لهذه الاتفاقية.
              </p>
            </div>
            <p v-else class="text-muted">
              يجب إضافة اتفاقية أتعاب أولاً لتتمكن من إضافة أو عرض أقساط.
            </p>
          </b-card-body>
        </b-collapse>
      </b-card>
      <!-- === نهاية قسم الأقساط المجدولة === -->

      <!-- === قسم الدفعات المستلمة === -->
      <b-card no-body class="mb-3">
        <b-card-header header-tag="header" class="p-1" role="tab">
          <b-button
            block
            v-b-toggle.collapse-payments
            variant="light"
            class="text-right"
          >
            <span class="float-left">
              <i class="fas fa-hand-holding-usd mr-2 text-success"></i>
              الدفعات المستلمة ({{
                financialData.payments ? financialData.payments.length : 0
              }})
            </span>
            <i class="fas fa-chevron-down"></i>
          </b-button>
        </b-card-header>
        <b-collapse
          id="collapse-payments"
          accordion="financial-accordion"
          role="tabpanel"
          visible
        >
          <b-card-body>
            <b-button
              size="sm"
              variant="success"
              class="mb-3"
              @click="openAddPaymentModal"
              :disabled="loading"
            >
              <i class="fas fa-plus"></i> تسجيل دفعة مستلمة جديدة
            </b-button>

            <!-- === جدول عرض الدفعات === -->
            <b-table
              v-if="financialData.payments && financialData.payments.length > 0"
              :items="financialData.payments"
              :fields="paymentsTableFields"
              striped
              hover
              responsive
              small
              bordered
              show-empty
              empty-text="لا توجد دفعات مستلمة لعرضها."
              class="mt-3"
              thead-class="table-light"
              tbody-tr-class="text-right"
            >
              <template #cell(payment_date)="data">
                {{ formatDate(data.value) }}
              </template>

              <template #cell(installment_id)="data">
                <span v-if="data.value">
                  <b-badge variant="info">القسط #{{ data.value }}</b-badge>
                </span>
                <span v-else class="text-muted">-</span>
              </template>

              <template #cell(payment_method)="data">
                {{ data.value || "-" }}
              </template>

              <template #cell(receipt)="{ item }">
                <b-link
                  v-if="item.receipt_document_id"
                  :href="`/api/documents/download_secure.php?doc_id=${item.receipt_document_id}`"
                  target="_blank"
                  class="btn btn-sm btn-info"
                  title="تحميل الإيصال"
                >
                  <i class="fas fa-receipt"></i> تحميل
                </b-link>
                <span v-else>-</span>
              </template>

              <template #cell(notes)="data">
                <span :title="data.value">{{
                  truncateText(data.value, 50)
                }}</span>
              </template>

              <template #cell(actions)="{ item }">
                <b-button
                  size="sm"
                  variant="outline-warning"
                  class="me-1"
                  @click="showEditPaymentModal(item)"
                  v-b-tooltip.hover
                  title="تعديل هذه الدفعة"
                >
                  <i class="fas fa-edit"></i>
                </b-button>
                <b-button
                  size="sm"
                  variant="outline-danger"
                  @click="confirmDeletePayment(item)"
                  :disabled="
                    isDeletingPayment && isDeletingPayment[item.payment_id]
                  "
                  v-b-tooltip.hover
                  title="حذف هذه الدفعة"
                >
                  <b-spinner
                    small
                    v-if="
                      isDeletingPayment && isDeletingPayment[item.payment_id]
                    "
                  ></b-spinner>
                  <i class="fas fa-trash-alt" v-else></i>
                </b-button>
              </template>
            </b-table>
            <p v-else class="text-muted">
              لم يتم تسجيل أي دفعات مستلمة لهذه القضية بعد.
            </p>
          </b-card-body>
        </b-collapse>
      </b-card>
      <!-- === نهاية قسم الدفعات المستلمة === -->

      <!-- قسم الملخص المالي القديم - تم نقله للأعلى -->
      <!--
      <b-card header="الملخص المالي" header-bg-variant="light" class="mb-3">
        <b-card-text>
          <p>
            <strong>إجمالي المستحق (المفوتر):</strong>
            <span class="text-muted"> (سيتم حسابه لاحقًا) </span>
          </p>
          <p>
            <strong>إجمالي المدفوع:</strong>
            {{ paymentsTotal | currency }}
          </p>
          <p>
            <strong>الرصيد الحالي:</strong>
            <span class="text-muted"> (سيتم حسابه لاحقًا) </span>
          </p>
        </b-card-text>
      </b-card>
      -->
    </div>

    <!-- === Modals === -->

    <!-- مودال إضافة اتفاقية أتعاب -->
    <b-modal
      id="add-agreement-modal"
      ref="addAgreementModalRef"
      title="إضافة اتفاقية أتعاب جديدة"
      size="lg"
      hide-footer
      @hidden="resetAddAgreementForm"
    >
      <b-alert
        variant="danger"
        :show="addAgreementError !== null"
        dismissible
        @dismissed="addAgreementError = null"
      >
        {{ addAgreementError }}
      </b-alert>
      <b-form @submit.prevent="submitAddAgreement">
        <b-form-group
          id="input-group-agreement-type"
          label="نوع الاتفاقية:"
          label-for="agreement-type"
        >
          <b-form-select
            id="agreement-type"
            v-model="newAgreement.agreement_type"
            :options="agreementTypeOptions"
            required
          ></b-form-select>
        </b-form-group>
        <b-form-group
          v-if="
            newAgreement.agreement_type === 'fixed' ||
            newAgreement.agreement_type === 'hybrid'
          "
          id="input-group-total-fixed"
          :label="
            newAgreement.agreement_type === 'fixed'
              ? 'المبلغ الإجمالي المقطوع:'
              : 'المبلغ المقدم الثابت:'
          "
          label-for="total-fixed-amount"
        >
          <b-form-input
            id="total-fixed-amount"
            v-model.number="newAgreement.total_fixed_amount"
            type="number"
            step="0.01"
            min="0"
            required
            placeholder="أدخل المبلغ"
          ></b-form-input>
        </b-form-group>
        <b-form-group
          v-if="
            newAgreement.agreement_type === 'percentage' ||
            newAgreement.agreement_type === 'hybrid'
          "
          id="input-group-percentage-rate"
          label="النسبة المئوية (%):"
          label-for="percentage-rate"
        >
          <b-form-input
            id="percentage-rate"
            v-model.number="newAgreement.percentage_rate"
            type="number"
            step="0.01"
            min="0.01"
            max="100"
            required
            placeholder="أدخل النسبة (مثال: 15)"
          ></b-form-input>
        </b-form-group>
        <b-form-group
          v-if="
            newAgreement.agreement_type === 'percentage' ||
            newAgreement.agreement_type === 'hybrid'
          "
          id="input-group-percentage-basis"
          label="أساس احتساب النسبة:"
          label-for="percentage-basis"
        >
          <b-form-textarea
            id="percentage-basis"
            v-model="newAgreement.contingency_trigger_description"
            required
            rows="3"
            placeholder="اشرح متى وكيف تستحق النسبة..."
          ></b-form-textarea>
        </b-form-group>
        <b-form-group
          id="input-group-agreement-date"
          label="تاريخ الاتفاقية:"
          label-for="agreement-date"
        >
          <b-form-datepicker
            id="agreement-date"
            v-model="newAgreement.agreement_date"
            required
            placeholder="اختر تاريخًا"
            locale="ar"
            :date-format-options="{
              year: 'numeric',
              month: 'numeric',
              day: 'numeric',
            }"
          ></b-form-datepicker>
        </b-form-group>
        <b-form-group id="input-group-notes" label="ملاحظات:" label-for="notes">
          <b-form-textarea
            id="notes"
            v-model="newAgreement.notes"
            rows="3"
          ></b-form-textarea>
        </b-form-group>
        <b-form-group
          id="input-group-contract-file"
          label="ملف العقد (اختياري - PDF فقط، 10MB كحد أقصى):"
          label-for="contract-file"
        >
          <b-form-file
            id="contract-file"
            ref="contractFileRef"
            v-model="newAgreement.contractFile"
            placeholder="اختر ملفًا أو اسحبه هنا..."
            drop-placeholder="اسحب الملف هنا..."
            accept=".pdf"
          ></b-form-file>
          <b-form-text v-if="newAgreement.contractFile">
            الملف المختار: {{ newAgreement.contractFile.name }}
          </b-form-text>
        </b-form-group>
        <hr />
        <div class="d-flex justify-content-end">
          <b-button
            type="submit"
            variant="success"
            :disabled="isSubmittingAgreement"
          >
            <b-spinner small v-if="isSubmittingAgreement"></b-spinner> حفظ
            الاتفاقية
          </b-button>
          <b-button
            variant="secondary"
            @click="hideAddAgreementModal"
            class="ms-2"
            >إلغاء</b-button
          >
        </div>
      </b-form>
    </b-modal>

    <!-- مودال تعديل اتفاقية الأتعاب -->
    <b-modal
      id="edit-agreement-modal"
      ref="editAgreementModalRef"
      title="تعديل تفاصيل اتفاقية الأتعاب"
      size="lg"
      hide-footer
      @hidden="resetEditAgreementForm"
    >
      <b-alert
        variant="danger"
        :show="editAgreementError !== null"
        dismissible
        @dismissed="editAgreementError = null"
      >
        {{ editAgreementError }}
      </b-alert>
      <b-form @submit.prevent="submitEditAgreement" v-if="editingAgreement">
        <b-form-group
          id="edit-input-group-agreement-type"
          label="نوع الاتفاقية:"
          label-for="edit-agreement-type"
        >
          <b-form-select
            id="edit-agreement-type"
            v-model="editingAgreement.agreement_type"
            :options="agreementTypeOptions"
            required
          ></b-form-select>
        </b-form-group>
        <b-form-group
          v-if="
            editingAgreement.agreement_type === 'fixed' ||
            editingAgreement.agreement_type === 'hybrid'
          "
          id="edit-input-group-total-fixed"
          :label="
            editingAgreement.agreement_type === 'fixed'
              ? 'المبلغ الإجمالي المقطوع:'
              : 'المبلغ المقدم الثابت:'
          "
          label-for="edit-total-fixed-amount"
        >
          <b-form-input
            id="edit-total-fixed-amount"
            v-model.number="editingAgreement.total_agreed_amount"
            type="number"
            step="0.01"
            min="0"
            required
            placeholder="أدخل المبلغ"
          ></b-form-input>
        </b-form-group>
        <b-form-group
          v-if="
            editingAgreement.agreement_type === 'percentage' ||
            editingAgreement.agreement_type === 'hybrid'
          "
          id="edit-input-group-percentage-rate"
          label="النسبة المئوية (%):"
          label-for="edit-percentage-rate"
        >
          <b-form-input
            id="edit-percentage-rate"
            v-model.number="editingAgreement.percentage_rate"
            type="number"
            step="0.01"
            min="0.01"
            max="100"
            required
            placeholder="أدخل النسبة (مثال: 15)"
          ></b-form-input>
        </b-form-group>
        <b-form-group
          v-if="
            editingAgreement.agreement_type === 'percentage' ||
            editingAgreement.agreement_type === 'hybrid'
          "
          id="edit-input-group-percentage-basis"
          label="أساس احتساب النسبة:"
          label-for="edit-percentage-basis"
        >
          <b-form-textarea
            id="edit-percentage-basis"
            v-model="editingAgreement.percentage_basis"
            required
            rows="3"
            placeholder="اشرح متى وكيف تستحق النسبة..."
          ></b-form-textarea>
        </b-form-group>
        <b-form-group
          id="edit-input-group-agreement-date"
          label="تاريخ الاتفاقية:"
          label-for="edit-agreement-date"
        >
          <b-form-datepicker
            id="edit-agreement-date"
            v-model="editingAgreement.agreement_date"
            placeholder="اختر تاريخًا"
            locale="ar"
            :date-format-options="{
              year: 'numeric',
              month: 'numeric',
              day: 'numeric',
            }"
          ></b-form-datepicker>
        </b-form-group>
        <b-form-group
          id="edit-input-group-notes"
          label="ملاحظات:"
          label-for="edit-notes"
        >
          <b-form-textarea
            id="edit-notes"
            v-model="editingAgreement.notes"
            rows="3"
          ></b-form-textarea>
        </b-form-group>

        <hr />
        <div class="d-flex justify-content-end">
          <b-button
            type="submit"
            variant="primary"
            :disabled="isSubmittingEdit"
          >
            <b-spinner small v-if="isSubmittingEdit"></b-spinner> حفظ التعديلات
          </b-button>
          <b-button
            variant="secondary"
            @click="hideEditAgreementModal"
            class="ms-2"
            >إلغاء</b-button
          >
        </div>
      </b-form>
      <div v-else class="text-center">
        <p>جاري تحميل بيانات الاتفاقية للتعديل...</p>
        <b-spinner></b-spinner>
      </div>
    </b-modal>

    <!-- Link Contract Modal -->
    <b-modal
      id="link-contract-modal"
      ref="linkContractModalRef"
      title="رفع وربط ملف العقد"
      size="md"
      hide-footer
      @hidden="resetLinkContractForm"
    >
      <b-alert
        variant="danger"
        :show="linkContractError !== null"
        dismissible
        @dismissed="linkContractError = null"
      >
        {{ linkContractError }}
      </b-alert>
      <b-form @submit.prevent="submitLinkContract">
        <b-form-group
          id="input-group-link-contract-file"
          label="ملف العقد (مطلوب - PDF فقط، 10MB كحد أقصى):"
          label-for="link-contract-file"
        >
          <b-form-file
            id="link-contract-file"
            ref="linkContractFileRef"
            v-model="contractToLinkFile"
            placeholder="اختر ملف PDF أو اسحبه هنا..."
            drop-placeholder="اسحب الملف هنا..."
            accept=".pdf"
            required
            :state="
              linkContractError && linkContractError.includes('File')
                ? false
                : null
            "
          ></b-form-file>
          <b-form-text v-if="contractToLinkFile">
            الملف المختار: {{ contractToLinkFile.name }}
          </b-form-text>
        </b-form-group>

        <hr />
        <div class="d-flex justify-content-end">
          <b-button
            type="submit"
            variant="success"
            :disabled="isSubmittingLinkContract || !contractToLinkFile"
          >
            <b-spinner small v-if="isSubmittingLinkContract"></b-spinner>
            {{ isSubmittingLinkContract ? "جاري الرفع..." : "رفع وربط العقد" }}
          </b-button>
          <b-button
            variant="secondary"
            @click="hideLinkContractModal"
            class="ms-2"
            >إلغاء</b-button
          >
        </div>
      </b-form>
    </b-modal>

    <!-- مودال إضافة قسط جديد -->
    <b-modal
      id="add-installment-modal"
      ref="addInstallmentModalRef"
      title="إضافة قسط مجدول جديد"
      size="lg"
      hide-footer
      @hidden="resetAddInstallmentForm"
    >
      <b-alert
        variant="danger"
        :show="addInstallmentError !== null"
        dismissible
        @dismissed="addInstallmentError = null"
      >
        {{ addInstallmentError }}
      </b-alert>
      <b-form @submit.prevent="submitAddInstallment">
        <b-form-group
          id="input-group-installment-amount"
          label="مبلغ القسط:"
          label-for="installment-amount"
        >
          <b-form-input
            id="installment-amount"
            v-model.number="newInstallment.amount_due"
            type="number"
            step="0.01"
            min="0.01"
            required
            placeholder="أدخل مبلغ القسط"
          ></b-form-input>
        </b-form-group>
        <b-form-group
          id="input-group-installment-trigger"
          label="شرط استحقاق القسط:"
          label-for="installment-trigger"
        >
          <b-form-select
            id="installment-trigger"
            v-model="newInstallment.due_trigger_type"
            :options="installmentTriggerTypeOptions"
            required
          ></b-form-select>
        </b-form-group>
        <b-form-group
          v-if="newInstallment.due_trigger_type === 'SPECIFIC_DATE'"
          id="input-group-installment-due-date"
          label="تاريخ الاستحقاق المحدد:"
          label-for="installment-due-date"
        >
          <b-form-datepicker
            id="installment-due-date"
            v-model="newInstallment.due_date"
            required
            placeholder="اختر تاريخًا"
            locale="ar"
            :date-format-options="{
              year: 'numeric',
              month: 'numeric',
              day: 'numeric',
            }"
          ></b-form-datepicker>
        </b-form-group>
        <b-form-group
          v-if="newInstallment.due_trigger_type === 'CUSTOM_EVENT_DESCRIPTION'"
          id="input-group-installment-event-desc"
          label="وصف الحدث المخصص للاستحقاق:"
          label-for="installment-event-desc"
        >
          <b-form-textarea
            id="installment-event-desc"
            v-model="newInstallment.due_event_description"
            required
            rows="3"
            placeholder="صف الحدث الذي يستحق عنده القسط (مثال: عند توقيع عقد التسوية)"
          ></b-form-textarea>
        </b-form-group>
        <b-form-group
          id="input-group-installment-notes"
          label="ملاحظات (اختياري):"
          label-for="installment-notes"
        >
          <b-form-textarea
            id="installment-notes"
            v-model="newInstallment.notes"
            rows="3"
          ></b-form-textarea>
        </b-form-group>
        <hr />
        <div class="d-flex justify-content-end">
          <b-button
            type="submit"
            variant="success"
            :disabled="isSubmittingInstallment"
          >
            <b-spinner small v-if="isSubmittingInstallment"></b-spinner> حفظ
            القسط
          </b-button>
          <b-button
            variant="secondary"
            @click="hideAddInstallmentModal"
            class="ms-2"
            >إلغاء</b-button
          >
        </div>
      </b-form>
    </b-modal>

    <!-- مودال تعديل قسط موجود -->
    <b-modal
      id="edit-installment-modal"
      ref="editInstallmentModalRef"
      title="تعديل القسط المجدول"
      size="lg"
      hide-footer
      @hidden="resetEditInstallmentForm"
    >
      <b-alert
        variant="danger"
        :show="editInstallmentError !== null"
        dismissible
        @dismissed="editInstallmentError = null"
      >
        {{ editInstallmentError }}
      </b-alert>
      <b-form @submit.prevent="submitEditInstallment" v-if="editingInstallment">
        <b-form-group
          id="edit-installment-amount-group"
          label="مبلغ القسط:"
          label-for="edit-installment-amount"
        >
          <b-form-input
            id="edit-installment-amount"
            v-model.number="editingInstallment.amount_due"
            type="number"
            step="0.01"
            min="0.01"
            required
            placeholder="أدخل مبلغ القسط"
          ></b-form-input>
        </b-form-group>
        <b-form-group
          id="edit-installment-trigger-group"
          label="شرط استحقاق القسط:"
          label-for="edit-installment-trigger"
        >
          <b-form-select
            id="edit-installment-trigger"
            v-model="editingInstallment.due_trigger_type"
            :options="installmentTriggerTypeOptions"
            required
          ></b-form-select>
        </b-form-group>
        <b-form-group
          v-if="editingInstallment.due_trigger_type === 'SPECIFIC_DATE'"
          id="edit-installment-due-date-group"
          label="تاريخ الاستحقاق المحدد:"
          label-for="edit-installment-due-date"
        >
          <b-form-datepicker
            id="edit-installment-due-date"
            v-model="editingInstallment.due_date"
            required
            placeholder="اختر تاريخًا"
            locale="ar"
            :date-format-options="{
              year: 'numeric',
              month: 'numeric',
              day: 'numeric',
            }"
          ></b-form-datepicker>
        </b-form-group>
        <b-form-group
          v-if="
            editingInstallment.due_trigger_type === 'CUSTOM_EVENT_DESCRIPTION'
          "
          id="edit-installment-event-desc-group"
          label="وصف الحدث المخصص للاستحقاق:"
          label-for="edit-installment-event-desc"
        >
          <b-form-textarea
            id="edit-installment-event-desc"
            v-model="editingInstallment.due_event_description"
            required
            rows="3"
            maxlength="100"
            placeholder="صف الحدث الذي يستحق عنده القسط (مثال: عند توقيع عقد التسوية)"
          ></b-form-textarea>
        </b-form-group>
        <b-form-group
          id="edit-installment-notes-group"
          label="ملاحظات (اختياري):"
          label-for="edit-installment-notes"
        >
          <b-form-textarea
            id="edit-installment-notes"
            v-model="editingInstallment.notes"
            rows="3"
          ></b-form-textarea>
        </b-form-group>
        <hr />
        <div class="d-flex justify-content-end">
          <b-button
            type="submit"
            variant="primary"
            :disabled="isSubmittingEditInstallment"
          >
            <b-spinner small v-if="isSubmittingEditInstallment"></b-spinner>
            حفظ التعديلات
          </b-button>
          <b-button
            variant="secondary"
            @click="hideEditInstallmentModal"
            class="ms-2"
            >إلغاء</b-button
          >
        </div>
      </b-form>
      <div v-else class="text-center">
        <p>جاري تحميل بيانات القسط للتعديل...</p>
        <b-spinner></b-spinner>
      </div>
    </b-modal>

    <!-- مودال تسجيل دفعة مستلمة -->
    <b-modal
      id="add-payment-modal"
      ref="addPaymentModalRef"
      title="تسجيل دفعة مستلمة جديدة"
      size="lg"
      @hidden="resetPaymentForm"
      hide-footer
    >
      <b-alert
        variant="danger"
        :show="!!paymentFormErrors && !!paymentFormErrors.general"
        dismissible
        @dismissed="paymentFormErrors = null"
      >
        <h5 class="alert-heading">خطأ!</h5>
        <p>{{ paymentFormErrors ? paymentFormErrors.general : "" }}</p>
      </b-alert>
      <b-alert
        variant="warning"
        :show="!!paymentFormErrors && !!paymentFormErrors.validation"
        dismissible
        @dismissed="paymentFormErrors = null"
      >
        <h5 class="alert-heading">الرجاء تصحيح الأخطاء التالية:</h5>
        <div v-if="paymentFormErrors && paymentFormErrors.validation">
          <ul v-if="Array.isArray(paymentFormErrors.validation)">
            <li
              v-for="(error, index) in paymentFormErrors.validation"
              :key="`val-err-${index}`"
            >
              {{ error }}
            </li>
          </ul>
          <p v-else>{{ paymentFormErrors.validation }}</p>
        </div>
      </b-alert>

      <b-form @submit.prevent="handlePaymentSubmit">
        <b-form-group
          label="المبلغ المستلم *"
          label-for="payment-amount"
          description="أدخل المبلغ الذي تم استلامه."
        >
          <b-form-input
            id="payment-amount"
            v-model.number="newPaymentData.amount_received"
            type="number"
            step="0.01"
            min="0.01"
            required
            placeholder="1000.00"
            :state="hasValidationError('amount_received') ? false : null"
          ></b-form-input>
        </b-form-group>

        <b-form-group label="تاريخ الدفع *" label-for="payment-date">
          <b-form-datepicker
            id="payment-date"
            v-model="newPaymentData.payment_date"
            required
            placeholder="اختر تاريخًا..."
            locale="ar-SA"
            :date-format-options="{
              year: 'numeric',
              month: '2-digit',
              day: '2-digit',
            }"
            :state="hasValidationError('payment_date') ? false : null"
          ></b-form-datepicker>
        </b-form-group>

        <b-form-group
          label="طريقة الدفع"
          label-for="payment-method"
          description="مثال: نقدي، تحويل بنكي، شيك رقم..."
        >
          <b-form-input
            id="payment-method"
            v-model.trim="newPaymentData.payment_method"
            type="text"
            placeholder="نقدي"
            :state="hasValidationError('payment_method') ? false : null"
          ></b-form-input>
        </b-form-group>

        <b-form-group
          label="ربط بقسط محدد (اختياري)"
          label-for="payment-installment-id"
          description="اختر القسط المجدول الذي تغطيه هذه الدفعة (إن وجد)."
        >
          <b-form-select
            id="payment-installment-id"
            v-model="newPaymentData.installment_id"
            :options="installmentOptions"
            value-field="value"
            text-field="text"
            :state="hasValidationError('installment_id') ? false : null"
          >
          </b-form-select>
        </b-form-group>

        <b-form-group label="ملاحظات إضافية" label-for="payment-notes">
          <b-form-textarea
            id="payment-notes"
            v-model="newPaymentData.notes"
            rows="3"
            max-rows="6"
            placeholder="أي تفاصيل إضافية حول الدفعة..."
            :state="hasValidationError('notes') ? false : null"
          ></b-form-textarea>
        </b-form-group>

        <b-form-group
          label="إرفاق إيصال (اختياري)"
          label-for="payment-receipt"
          description="يمكنك إرفاق ملف الإيصال (PDF, JPG, PNG, GIF - حد أقصى 5MB)."
        >
          <b-form-file
            id="payment-receipt"
            ref="paymentFileInput"
            v-model="newPaymentData.receipt_document"
            placeholder="اختر ملفًا أو اسحبه هنا..."
            drop-placeholder="اسحب الملف هنا..."
            accept=".pdf, .jpg, .jpeg, .png, .gif"
            :state="
              hasValidationError('receipt') || hasValidationError('File')
                ? false
                : null
            "
          ></b-form-file>
        </b-form-group>

        <hr />

        <div class="d-flex justify-content-end">
          <b-button
            type="button"
            variant="secondary"
            @click="$bvModal.hide('add-payment-modal')"
            :disabled="isSubmittingPayment"
            class="ms-2"
            >إلغاء</b-button
          >
          <b-button
            type="submit"
            variant="success"
            :disabled="isSubmittingPayment"
          >
            <b-spinner small v-if="isSubmittingPayment"></b-spinner>
            {{ isSubmittingPayment ? "جاري الحفظ..." : "حفظ الدفعة" }}
          </b-button>
        </div>
      </b-form>
    </b-modal>

    <!-- *** مودال تعديل دفعة مستلمة *** -->
    <b-modal
      id="edit-payment-modal"
      ref="editPaymentModalRef"
      title="تعديل الدفعة المستلمة"
      size="lg"
      @hidden="resetEditPaymentForm"
      hide-footer
    >
      <b-alert
        variant="danger"
        :show="!!editPaymentError"
        dismissible
        @dismissed="editPaymentError = null"
      >
        {{ editPaymentError }}
      </b-alert>

      <b-form @submit.prevent="submitEditPayment" v-if="editingPayment">
        <p class="text-muted">معرف الدفعة: {{ editingPayment.payment_id }}</p>

        <b-form-group
          label="المبلغ المستلم *"
          label-for="edit-payment-amount"
          description="أدخل المبلغ الذي تم استلامه."
        >
          <b-form-input
            id="edit-payment-amount"
            v-model.number="editingPayment.amount_received"
            type="number"
            step="0.01"
            min="0.01"
            required
            placeholder="1000.00"
          ></b-form-input>
        </b-form-group>

        <b-form-group label="تاريخ الدفع *" label-for="edit-payment-date">
          <b-form-datepicker
            id="edit-payment-date"
            v-model="editingPayment.payment_date"
            required
            placeholder="اختر تاريخًا..."
            locale="ar-SA"
            :date-format-options="{
              year: 'numeric',
              month: '2-digit',
              day: '2-digit',
            }"
          ></b-form-datepicker>
        </b-form-group>

        <b-form-group
          label="طريقة الدفع"
          label-for="edit-payment-method"
          description="مثال: نقدي، تحويل بنكي، شيك رقم..."
        >
          <b-form-input
            id="edit-payment-method"
            v-model.trim="editingPayment.payment_method"
            type="text"
            placeholder="نقدي"
          ></b-form-input>
        </b-form-group>

        <b-form-group
          label="ربط بقسط محدد (اختياري)"
          label-for="edit-payment-installment-id"
          description="اختر القسط المجدول الذي تغطيه هذه الدفعة (إن وجد)."
        >
          <b-form-select
            id="edit-payment-installment-id"
            v-model="editingPayment.installment_id"
            :options="installmentOptions"
            value-field="value"
            text-field="text"
          >
          </b-form-select>
        </b-form-group>

        <b-form-group label="ملاحظات إضافية" label-for="edit-payment-notes">
          <b-form-textarea
            id="edit-payment-notes"
            v-model="editingPayment.notes"
            rows="3"
            max-rows="6"
            placeholder="أي تفاصيل إضافية حول الدفعة..."
          ></b-form-textarea>
        </b-form-group>

        <b-form-group
          label="إيصال الدفع"
          label-for="edit-payment-receipt"
          description="يمكنك رفع إيصال جديد ليحل محل الحالي، أو إزالة الإيصال الحالي."
        >
          <div
            v-if="currentReceiptDetails && !removeCurrentReceipt"
            class="mb-2 alert alert-info py-1"
          >
            <i class="fas fa-receipt me-1"></i>
            يوجد إيصال مرفق حاليًا.
            <b-link
              :href="`/api/documents/download_secure.php?doc_id=${currentReceiptDetails.id}`"
              target="_blank"
              class="btn btn-sm btn-outline-info ms-2"
              title="تحميل الإيصال الحالي"
            >
              <i class="fas fa-download"></i> تحميل
            </b-link>
            <b-form-checkbox
              v-model="removeCurrentReceipt"
              class="d-inline-block ms-3"
              :disabled="!!editPaymentReceiptFile"
            >
              حذف الإيصال الحالي عند الحفظ
            </b-form-checkbox>
          </div>
          <div v-else-if="currentReceiptDetails && removeCurrentReceipt">
            <p class="text-warning">
              <i class="fas fa-exclamation-triangle me-1"></i> سيتم حذف الإيصال
              الحالي عند الحفظ.
            </p>
          </div>
          <div v-else class="mb-2">
            <p class="text-muted">لا يوجد إيصال مرفق حاليًا.</p>
          </div>

          <b-form-file
            id="edit-payment-receipt"
            ref="editPaymentFileInput"
            v-model="editPaymentReceiptFile"
            placeholder="اختر ملف إيصال جديد (اختياري)..."
            drop-placeholder="اسحب الملف هنا..."
            accept=".pdf, .jpg, .jpeg, .png, .gif"
            :disabled="removeCurrentReceipt"
          ></b-form-file>
          <b-form-text v-if="editPaymentReceiptFile" class="mt-1 text-success">
            <i class="fas fa-check me-1"></i>
            تم اختيار ملف جديد: {{ editPaymentReceiptFile.name }} (سيحل محل
            الإيصال الحالي إن وجد).
          </b-form-text>
        </b-form-group>

        <hr />

        <div class="d-flex justify-content-end">
          <b-button
            type="button"
            variant="secondary"
            @click="hideEditPaymentModal"
            :disabled="isSubmittingEditPayment"
            class="ms-2"
            >إلغاء</b-button
          >
          <b-button
            type="submit"
            variant="primary"
            :disabled="isSubmittingEditPayment"
          >
            <b-spinner small v-if="isSubmittingEditPayment"></b-spinner>
            {{ isSubmittingEditPayment ? "جاري الحفظ..." : "حفظ التعديلات" }}
          </b-button>
        </div>
      </b-form>
      <div v-else class="text-center">
        <p>جاري تحميل بيانات الدفعة للتعديل...</p>
        <b-spinner></b-spinner>
      </div>
    </b-modal>
  </div>
</template>
<script>
import axios from "axios";

// Default objects
const defaultNewAgreement = {
  agreement_type: null,
  total_fixed_amount: null, // Note: Renamed from total_agreed_amount in form for clarity
  percentage_rate: null,
  contingency_trigger_description: "", // Note: Mapped to percentage_basis in API
  agreement_date: null,
  notes: "",
  contractFile: null,
};

const defaultNewInstallment = {
  amount_due: null,
  due_trigger_type: null,
  due_date: null,
  due_event_description: "",
  notes: "",
};

const defaultEditingInstallment = null;

const defaultNewPayment = {
  amount_received: null,
  payment_date: null,
  payment_method: "",
  installment_id: null,
  notes: "",
  receipt_document: null, // Used for file input v-model
};

const defaultEditingPayment = null;

// Default Toast Options
const defaultToastOptions = {
  toaster: "b-toaster-top-center",
  autoHideDelay: 8000,
  solid: true,
  appendToast: true,
};

export default {
  name: "CaseFinancialsView",
  props: { id: { type: [String, Number], required: true } },
  data() {
    return {
      // General state
      loading: false,
      error: null,
      financialData: {
        agreement: null,
        installments: [],
        payments: [],
        contract_document: null,
        // Removed summary from here as it will be computed
      },
      caseInfo: null,

      // Add Agreement Modal State
      newAgreement: { ...defaultNewAgreement },
      agreementTypeOptions: [
        { value: null, text: "يرجى اختيار نوع الاتفاقية", disabled: true },
        { value: "fixed", text: "مبلغ مقطوع" },
        { value: "percentage", text: "نسبة مئوية فقط" },
        { value: "hybrid", text: "مقدم ثابت + نسبة" },
      ],
      addAgreementError: null,
      isSubmittingAgreement: false,

      // Edit Agreement Modal State
      editingAgreement: null, // This will hold the agreement object for editing
      editAgreementError: null,
      isSubmittingEdit: false,

      // Link Contract Modal State
      contractToLinkFile: null,
      linkContractError: null,
      isSubmittingLinkContract: false,

      // State for deleting agreement
      isDeletingAgreement: false,

      // Add Installment Modal State
      newInstallment: { ...defaultNewInstallment },
      installmentTriggerTypeOptions: [
        { value: null, text: "يرجى اختيار شرط الاستحقاق", disabled: true },
        { value: "SPECIFIC_DATE", text: "تاريخ محدد" },
        { value: "INITIAL_RULING_ISSUED", text: "عند صدور حكم ابتدائي" },
        { value: "APPEAL_RULING_ISSUED", text: "عند صدور حكم استئناف" },
        { value: "SUPREME_RULING_ISSUED", text: "عند صدور قرار محكمة عليا" },
        { value: "CUSTOM_EVENT_DESCRIPTION", text: "حدث مخصص (يتطلب وصف)" },
      ],
      addInstallmentError: null,
      isSubmittingInstallment: false,

      // Edit Installment Modal State
      editingInstallment: defaultEditingInstallment, // Will hold the installment object for editing
      editInstallmentError: null,
      isSubmittingEditInstallment: false,

      // State for changing installment status
      isChangingStatus: {}, // Use object { installment_id: boolean }

      // State for deleting installment
      isDeletingInstallment: {}, // Use object { installment_id: boolean }

      // Installment Table Fields
      installmentTableFields: [
        {
          key: "amount_due",
          label: "المبلغ",
          class: "text-center",
          sortable: true,
        },
        { key: "due_trigger_type", label: "شرط الاستحقاق", sortable: false },
        { key: "due_details", label: "تاريخ / وصف الاستحقاق", sortable: false },
        {
          key: "status",
          label: "الحالة",
          class: "text-center status-col",
          sortable: true,
        },
        { key: "notes", label: "ملاحظات", sortable: false },
        {
          key: "actions",
          label: "إجراءات",
          class: "text-center actions-col",
          sortable: false,
        },
      ],

      // Payments Table Fields
      paymentsTableFields: [
        {
          key: "payment_date",
          label: "تاريخ الدفع",
          class: "text-center",
          sortable: true,
        },
        {
          key: "amount_received",
          label: "المبلغ المستلم",
          class: "text-center",
          sortable: true,
          formatter: (value) => this.$options.filters.currency(value),
        },
        { key: "payment_method", label: "طريقة الدفع", sortable: false },
        {
          key: "installment_id",
          label: "القسط المرتبط",
          class: "text-center",
          sortable: false,
        },
        {
          key: "receipt",
          label: "الإيصال",
          class: "text-center",
          sortable: false,
        },
        { key: "notes", label: "ملاحظات", sortable: false },
        {
          key: "actions",
          label: "إجراءات",
          class: "text-center actions-col",
          sortable: false,
        },
      ],

      // Add Payment Modal State
      newPaymentData: { ...defaultNewPayment, case_id: this.id },
      isSubmittingPayment: false,
      paymentFormErrors: null, // { general: '...', validation: { field: '...' } }

      // Edit Payment Modal State
      editingPayment: defaultEditingPayment, // Will hold the payment object for editing
      editPaymentError: null,
      isSubmittingEditPayment: false,
      editPaymentReceiptFile: null, // v-model for the file input in edit modal
      removeCurrentReceipt: false, // checkbox v-model in edit modal
      currentReceiptDetails: null, // { id: document_id } - to show current receipt info

      // State for deleting payment
      isDeletingPayment: {}, // Use object { payment_id: boolean }

      // Manually allowed statuses for dropdown change (exclude 'paid')
      allowedManualStatuses: ["pending", "due", "overdue", "cancelled"],
    };
  },
  computed: {
    installmentOptions() {
      const options = [
        { value: null, text: "دفعة عامة (غير مرتبطة بقسط محدد)" },
      ];
      const installments = this.financialData?.installments ?? [];
      installments.forEach((inst) => {
        // Only include installments that are not cancelled or already paid?
        // if (inst.status !== 'cancelled' && inst.status !== 'paid') {
        let dueDateText = "";
        if (inst.due_trigger_type === "SPECIFIC_DATE" && inst.due_date) {
          dueDateText = `تاريخ: ${this.formatDate(inst.due_date)}`;
        } else {
          dueDateText = `شرط: ${this.getTriggerTypeText(
            inst.due_trigger_type
          )}`;
        }
        options.push({
          value: inst.installment_id,
          text: `قسط #${inst.installment_id} (${this.$options.filters.currency(
            inst.amount_due
          )}) - ${dueDateText} - الحالة: ${this.getStatusText(inst.status)}`,
        });
        // }
      });
      return options;
    },
    paymentsTotal() {
      return (this.financialData?.payments ?? []).reduce(
        (sum, payment) => sum + parseFloat(payment.amount_received || 0),
        0
      );
    },

    // *** START: New Computed Properties for Financial Summary ***

    /**
     * Calculates the total amount due (billed) based on the fee agreement.
     * Follows the logic defined in the SSOT.
     * @returns {number} The total due/billed amount.
     */
    totalDueBilled() {
      // Guard clause: No data or no agreement means nothing is billed yet.
      if (!this.financialData || !this.financialData.agreement) {
        return 0;
      }

      const agreement = this.financialData.agreement;
      const installments = this.financialData.installments || []; // Ensure installments is an array

      try {
        switch (agreement.agreement_type) {
          case "fixed":
            // For fixed type, use the total_agreed_amount.
            // Use parseFloat to ensure it's a number, default to 0 if null/invalid.
            return parseFloat(agreement.total_agreed_amount) || 0;

          case "percentage":
          case "hybrid":
            // For percentage or hybrid, sum the amount_due of non-cancelled installments.
            return installments.reduce((total, installment) => {
              // Check if the installment is not cancelled and has a valid amount_due.
              if (
                installment.status !== "cancelled" &&
                installment.amount_due
              ) {
                // Add the parsed amount to the total. Default to 0 if parsing fails.
                return total + (parseFloat(installment.amount_due) || 0);
              }
              // If cancelled or no amount, just return the current total.
              return total;
            }, 0); // Start the sum from 0.

          default:
            // Handle unknown or unsupported agreement types.
            console.warn(
              `[Computed:totalDueBilled] Unknown agreement type for calculation: ${agreement.agreement_type}`
            );
            return 0; // Return 0 for unknown types as per SSOT logic.
        }
      } catch (error) {
        // Catch any unexpected errors during calculation.
        console.error(
          "[Computed:totalDueBilled] Error calculating totalDueBilled:",
          error
        );
        return 0; // Return 0 in case of errors.
      }
    },

    /**
     * Calculates the current balance (remaining amount to be paid).
     * It's the difference between the total due/billed and the total payments received.
     * @returns {number} The current balance. Can be negative if overpaid.
     */
    currentBalance() {
      // Ensure both totalDueBilled and paymentsTotal are numbers before subtraction.
      // Access them via `this.` as they are computed properties themselves.
      const due =
        typeof this.totalDueBilled === "number" ? this.totalDueBilled : 0;
      const paid =
        typeof this.paymentsTotal === "number" ? this.paymentsTotal : 0;

      // Calculate the difference.
      return due - paid;
    },

    // *** END: New Computed Properties for Financial Summary ***
  },
  watch: {
    // Watcher to clear the 'remove receipt' checkbox if a new file is selected
    editPaymentReceiptFile(newFile) {
      if (newFile instanceof File) {
        this.removeCurrentReceipt = false;
      }
    },
    // Watcher to clear the file input if 'remove receipt' checkbox is checked
    removeCurrentReceipt(wantsToRemove) {
      if (wantsToRemove) {
        this.editPaymentReceiptFile = null;
        // Reset the file input visually using its ref
        if (this.$refs.editPaymentFileInput) {
          this.$refs.editPaymentFileInput.reset();
        }
      }
    },
  },
  created() {
    this.fetchFinancialData();
  },
  methods: {
    async fetchFinancialData() {
      this.loading = true;
      this.error = null;
      try {
        const response = await axios.get(
          `/api/financials/get_case_financials.php?case_id=${this.id}`,
          { withCredentials: true }
        );
        if (response.status === 200 && response.data?.success) {
          const data = response.data.data || {};
          this.financialData = {
            agreement: data.agreement || null,
            installments: data.installments || [],
            payments: data.payments_received || [], // Ensure key matches API response
            contract_document: data.contract_document || null,
            // Removed summary as it's now computed
          };
          this.caseInfo = data.case_info || null; // Store case info if provided
        } else {
          // Handle API error response (e.g., success: false)
          this.error = response.data?.message || "حدث خطأ غير متوقع.";
          this.resetFinancialDataOnError(); // Reset data on error
        }
      } catch (err) {
        // Handle network/request errors
        console.error("Fetch Financial Data Error:", err);
        this.error = this.getErrorMessage(err);
        this.resetFinancialDataOnError(); // Reset data on error
      } finally {
        this.loading = false;
      }
    },

    // Resets financial data and related editing states on fetch error or deletion
    resetFinancialDataOnError() {
      this.financialData = {
        agreement: null,
        installments: [],
        payments: [],
        contract_document: null,
      };
      // Reset modal/editing states as well
      this.editingAgreement = null;
      this.editingInstallment = null;
      this.isChangingStatus = {};
      this.isDeletingInstallment = {};
      this.editingPayment = null;
      this.editPaymentError = null;
      this.isSubmittingEditPayment = false;
      this.editPaymentReceiptFile = null;
      this.removeCurrentReceipt = false;
      this.currentReceiptDetails = null;
      this.isDeletingPayment = {};
      this.isDeletingAgreement = false; // Reset agreement deletion state
    },

    // Centralized error message handling
    getErrorMessage(err) {
      if (err.response) {
        // Prioritize message from API response body
        let msg =
          err.response.data && err.response.data.message
            ? err.response.data.message
            : `خطأ ${err.response.status}: ${err.response.statusText}`;

        // Specific status code handling
        if (err.response.status === 401) msg = "غير مصرح لك.";
        else if (err.response.status === 403)
          msg = "غير مصرح لك بالوصول لهذه البيانات.";
        else if (err.response.status === 404)
          msg = `لم يتم العثور على نقطة نهاية API (${
            err.config?.url || ""
          }). تأكد من تشغيل الواجهة الخلفية وإعداد الوكيل.`;
        else if (err.response.status === 405)
          msg = `الطريقة (${
            err.config?.method?.toUpperCase() || ""
          }) غير مسموحة لنقطة النهاية (${err.config?.url || ""}).`;
        else if (err.response.status === 500)
          msg = `خطأ داخلي في الخادم: ${
            err.response.data?.message || "فشل العملية"
          }`;
        else if (err.response.status === 413)
          msg = "حجم الملف يتجاوز الحد المسموح به.";
        else if (err.response.status === 415) msg = "نوع الملف غير مدعوم.";
        else if (err.response.status === 409)
          // Conflict (e.g., dependency constraint)
          msg =
            err.response.data?.message ||
            "تعذر إكمال العملية بسبب تعارض في البيانات (قد تكون مرتبطة ببيانات أخرى).";
        else if (err.response.status === 400 && err.response.data?.errors) {
          // Validation errors
          msg =
            "فشل التحقق من صحة البيانات: " +
            this.formatApiError(err.response.data);
        }
        // Ensure API message is used if available for other errors too
        else if (err.response.data?.message) {
          msg = err.response.data.message;
        }
        return msg;
      } else if (err.request) {
        // Network error (no response received)
        return "لا يمكن الوصول إلى الخادم. تحقق من الشبكة وتشغيل الواجهة الخلفية والوكيل.";
      } else {
        // Frontend error (error setting up request)
        return `حدث خطأ غير متوقع في الواجهة الأمامية: ${err.message}`;
      }
    },

    // Helper to get display text for agreement type
    getAgreementTypeText(agreementType) {
      switch (agreementType) {
        case "fixed":
          return "مبلغ مقطوع";
        case "percentage":
          return "نسبة مئوية فقط";
        case "hybrid":
          return "مقدم ثابت + نسبة";
        default:
          return agreementType || "غير محدد";
      }
    },

    // --- Add Agreement Methods ---
    showAddAgreementModal() {
      this.resetAddAgreementForm();
      this.$bvModal.show("add-agreement-modal");
    },
    hideAddAgreementModal() {
      this.$bvModal.hide("add-agreement-modal");
    },
    resetAddAgreementForm() {
      this.newAgreement = { ...defaultNewAgreement };
      this.addAgreementError = null;
      if (this.$refs.contractFileRef) this.$refs.contractFileRef.reset();
      this.isSubmittingAgreement = false;
    },
    async submitAddAgreement() {
      this.isSubmittingAgreement = true;
      this.addAgreementError = null;
      const formData = new FormData();

      // Append mandatory fields
      formData.append("case_id", this.id);
      formData.append("agreement_type", this.newAgreement.agreement_type);
      if (this.newAgreement.agreement_date) {
        formData.append("agreement_date", this.newAgreement.agreement_date);
      }

      // Append conditional fields based on type
      if (["fixed", "hybrid"].includes(this.newAgreement.agreement_type)) {
        formData.append(
          "total_agreed_amount",
          this.newAgreement.total_fixed_amount
        ); // Map form field to API field
      }
      if (["percentage", "hybrid"].includes(this.newAgreement.agreement_type)) {
        formData.append("percentage_rate", this.newAgreement.percentage_rate);
        formData.append(
          "percentage_basis",
          this.newAgreement.contingency_trigger_description
        ); // Map form field to API field
      }

      // Append optional fields
      if (this.newAgreement.notes) {
        formData.append("notes", this.newAgreement.notes);
      }
      if (this.newAgreement.contractFile) {
        formData.append("contractFile", this.newAgreement.contractFile);
      }

      try {
        const response = await axios.post(
          "/api/financials/add_fee_agreement.php",
          formData,
          {
            headers: { "Content-Type": "multipart/form-data" },
            withCredentials: true,
          }
        );
        if (response.data?.success) {
          this.hideAddAgreementModal();
          this.$bvToast.toast("تمت إضافة اتفاقية الأتعاب بنجاح.", {
            title: "نجاح",
            variant: "success",
            ...defaultToastOptions,
          });
          this.fetchFinancialData(); // Refresh data
        } else {
          this.addAgreementError = this.formatApiError(response.data);
        }
      } catch (err) {
        console.error("Error submitting agreement:", err);
        this.addAgreementError = this.getErrorMessage(err);
      } finally {
        this.isSubmittingAgreement = false;
      }
    },

    // --- Edit Agreement Methods ---
    showEditAgreementModal() {
      if (!this.financialData.agreement) {
        this.$bvToast.toast("لا توجد اتفاقية لتعديلها.", {
          title: "خطأ",
          variant: "danger",
          ...defaultToastOptions,
        });
        return;
      }
      // Deep clone the agreement object to avoid modifying the original data directly
      this.editingAgreement = JSON.parse(
        JSON.stringify(this.financialData.agreement)
      );
      // Pre-populate date picker correctly
      if (this.editingAgreement.agreement_date) {
        try {
          const date = new Date(this.editingAgreement.agreement_date);
          if (!isNaN(date.getTime())) {
            // Adjust for timezone offset to display correctly in datepicker
            const timezoneOffset = date.getTimezoneOffset() * 60000;
            const adjustedDate = new Date(date.getTime() - timezoneOffset);
            this.editingAgreement.agreement_date = adjustedDate
              .toISOString()
              .split("T")[0];
          } else {
            this.editingAgreement.agreement_date = null; // Clear invalid date
          }
        } catch (e) {
          console.error("Error parsing agreement date for edit:", e);
          this.editingAgreement.agreement_date = null;
        }
      }

      this.editAgreementError = null;
      this.isSubmittingEdit = false;
      this.$bvModal.show("edit-agreement-modal");
    },
    hideEditAgreementModal() {
      this.$bvModal.hide("edit-agreement-modal");
    },
    resetEditAgreementForm() {
      this.editingAgreement = null; // Reset the editing object
      this.editAgreementError = null;
      this.isSubmittingEdit = false;
    },
    async submitEditAgreement() {
      if (!this.editingAgreement || !this.editingAgreement.agreement_id) {
        this.editAgreementError =
          "خطأ: لا يمكن تحديد الاتفاقية المراد تعديلها.";
        return;
      }
      this.isSubmittingEdit = true;
      this.editAgreementError = null;

      // Prepare payload based on the editingAgreement object
      const payload = {
        agreement_id: this.editingAgreement.agreement_id,
        agreement_type: this.editingAgreement.agreement_type,
        total_agreed_amount: ["fixed", "hybrid"].includes(
          this.editingAgreement.agreement_type
        )
          ? this.editingAgreement.total_agreed_amount
          : null,
        percentage_rate: ["percentage", "hybrid"].includes(
          this.editingAgreement.agreement_type
        )
          ? this.editingAgreement.percentage_rate
          : null,
        percentage_basis: ["percentage", "hybrid"].includes(
          this.editingAgreement.agreement_type
        )
          ? this.editingAgreement.percentage_basis // API expects percentage_basis
          : null,
        agreement_date: this.editingAgreement.agreement_date || null,
        notes: this.editingAgreement.notes || null,
      };

      try {
        const response = await axios.post(
          "/api/financials/update_fee_agreement.php",
          payload,
          {
            headers: { "Content-Type": "application/json" },
            withCredentials: true,
          }
        );

        if (response.data?.success) {
          this.hideEditAgreementModal();
          this.$bvToast.toast(
            response.data.message || "تم تحديث تفاصيل الاتفاقية بنجاح.",
            { title: "نجاح", variant: "success", ...defaultToastOptions }
          );
          this.fetchFinancialData(); // Refresh data
        } else {
          this.editAgreementError = this.formatApiError(response.data);
        }
      } catch (err) {
        console.error("Error submitting agreement edit:", err);
        this.editAgreementError = this.getErrorMessage(err);
        // Optionally display validation errors more specifically if needed
        if (err.response?.status === 400 && err.response.data?.errors) {
          this.editAgreementError =
            "فشل التحقق من صحة البيانات: \n" +
            this.formatApiError(err.response.data);
        }
      } finally {
        this.isSubmittingEdit = false;
      }
    },

    // --- Link Contract Methods ---
    showLinkContractModal() {
      if (!this.financialData?.agreement?.agreement_id) {
        this.$bvToast.toast("لا يمكن ربط العقد لعدم وجود اتفاقية أتعاب.", {
          title: "خطأ",
          variant: "warning",
          ...defaultToastOptions,
        });
        return;
      }
      if (this.financialData.contract_document) {
        this.$bvToast.toast(
          "يوجد عقد مرفق بالفعل. قم بحذفه أولاً لرفع عقد جديد.",
          {
            title: "تنبيه",
            variant: "info",
            ...defaultToastOptions,
          }
        );
        return;
      }
      this.resetLinkContractForm();
      this.$bvModal.show("link-contract-modal");
    },
    hideLinkContractModal() {
      this.$bvModal.hide("link-contract-modal");
    },
    resetLinkContractForm() {
      this.contractToLinkFile = null;
      this.linkContractError = null;
      this.isSubmittingLinkContract = false;
      if (this.$refs.linkContractFileRef)
        this.$refs.linkContractFileRef.reset();
    },
    async submitLinkContract() {
      if (!this.financialData?.agreement?.agreement_id) {
        this.linkContractError = "خطأ: معرف الاتفاقية غير موجود.";
        return;
      }
      if (!this.contractToLinkFile) {
        this.linkContractError = "الرجاء اختيار ملف العقد (PDF) للرفع.";
        return;
      }
      this.isSubmittingLinkContract = true;
      this.linkContractError = null;
      const formData = new FormData();
      formData.append(
        "agreement_id",
        this.financialData.agreement.agreement_id
      );
      formData.append("contractFile", this.contractToLinkFile);

      try {
        const response = await axios.post(
          "/api/financials/link_agreement_contract.php",
          formData,
          {
            headers: { "Content-Type": "multipart/form-data" },
            withCredentials: true,
          }
        );

        if (response.data?.success) {
          this.hideLinkContractModal();
          this.$bvToast.toast(
            response.data.message || "تم رفع وربط ملف العقد بنجاح.",
            { title: "نجاح", variant: "success", ...defaultToastOptions }
          );
          this.fetchFinancialData(); // Refresh to show the linked contract
        } else {
          this.linkContractError = this.formatApiError(response.data);
        }
      } catch (err) {
        console.error("Error linking contract:", err);
        this.linkContractError = this.getErrorMessage(err);
      } finally {
        this.isSubmittingLinkContract = false;
      }
    },

    // --- Add Installment Methods ---
    showAddInstallmentModal() {
      if (!this.financialData?.agreement?.agreement_id) {
        this.$bvToast.toast("يجب وجود اتفاقية أتعاب أولاً لإضافة قسط.", {
          title: "تنبيه",
          variant: "warning",
          ...defaultToastOptions,
        });
        return;
      }
      this.resetAddInstallmentForm();
      this.$bvModal.show("add-installment-modal");
    },
    hideAddInstallmentModal() {
      this.$bvModal.hide("add-installment-modal");
    },
    resetAddInstallmentForm() {
      this.newInstallment = { ...defaultNewInstallment };
      this.addInstallmentError = null;
      this.isSubmittingInstallment = false;
    },
    async submitAddInstallment() {
      this.isSubmittingInstallment = true;
      this.addInstallmentError = null;
      if (!this.financialData?.agreement?.agreement_id) {
        this.addInstallmentError = "خطأ: لا يمكن تحديد اتفاقية الأتعاب.";
        this.isSubmittingInstallment = false;
        return;
      }
      // Prepare payload ensuring null for conditional fields if not applicable
      const payload = {
        agreement_id: this.financialData.agreement.agreement_id,
        amount_due: this.newInstallment.amount_due,
        due_trigger_type: this.newInstallment.due_trigger_type,
        due_date:
          this.newInstallment.due_trigger_type === "SPECIFIC_DATE"
            ? this.newInstallment.due_date
            : null,
        due_event_description:
          this.newInstallment.due_trigger_type === "CUSTOM_EVENT_DESCRIPTION"
            ? this.newInstallment.due_event_description
            : null,
        notes: this.newInstallment.notes || null,
      };
      try {
        const response = await axios.post(
          "/api/financials/add_fee_installment.php",
          payload,
          {
            headers: { "Content-Type": "application/json" },
            withCredentials: true,
          }
        );
        if (response.status === 201 && response.data?.success) {
          this.hideAddInstallmentModal();
          this.$bvToast.toast("تمت إضافة القسط المجدول بنجاح.", {
            title: "نجاح",
            variant: "success",
            ...defaultToastOptions,
          });
          this.fetchFinancialData(); // Refresh data
        } else {
          this.addInstallmentError = this.formatApiError(response.data);
        }
      } catch (err) {
        console.error("Error submitting installment:", err);
        this.addInstallmentError = this.getErrorMessage(err);
      } finally {
        this.isSubmittingInstallment = false;
      }
    },

    // --- Edit Installment Methods ---
    showEditInstallmentModal(installmentItem) {
      if (!installmentItem || !installmentItem.installment_id) {
        console.error(
          "Invalid installment object passed to edit modal:",
          installmentItem
        );
        this.$bvToast.toast("خطأ: بيانات القسط غير صالحة للتعديل.", {
          title: "خطأ داخلي",
          variant: "danger",
          ...defaultToastOptions,
        });
        return;
      }
      // Deep clone to avoid modifying original data
      this.editingInstallment = JSON.parse(JSON.stringify(installmentItem));
      // Format date correctly for datepicker (handle timezone offset)
      if (this.editingInstallment.due_date) {
        try {
          const date = new Date(this.editingInstallment.due_date);
          if (!isNaN(date.getTime())) {
            const timezoneOffset = date.getTimezoneOffset() * 60000;
            const adjustedDate = new Date(date.getTime() - timezoneOffset);
            this.editingInstallment.due_date = adjustedDate
              .toISOString()
              .split("T")[0];
          } else {
            console.warn(
              "Invalid date format in installment data:",
              installmentItem.due_date
            );
            this.editingInstallment.due_date = null; // Clear if invalid
          }
        } catch (e) {
          console.error("Error parsing installment due date:", e);
          this.editingInstallment.due_date = null;
        }
      }
      this.editInstallmentError = null;
      this.isSubmittingEditInstallment = false;
      this.$bvModal.show("edit-installment-modal");
    },
    hideEditInstallmentModal() {
      this.$bvModal.hide("edit-installment-modal");
    },
    resetEditInstallmentForm() {
      this.editingInstallment = defaultEditingInstallment; // Reset to null
      this.editInstallmentError = null;
      this.isSubmittingEditInstallment = false;
    },
    async submitEditInstallment() {
      if (!this.editingInstallment || !this.editingInstallment.installment_id) {
        this.editInstallmentError = "خطأ: لا يمكن تحديد القسط المراد تعديله.";
        return;
      }
      this.isSubmittingEditInstallment = true;
      this.editInstallmentError = null;

      // Prepare payload, ensuring conditional fields are null if not applicable
      const payload = {
        installment_id: this.editingInstallment.installment_id,
        amount_due: this.editingInstallment.amount_due,
        due_trigger_type: this.editingInstallment.due_trigger_type,
        due_date:
          this.editingInstallment.due_trigger_type === "SPECIFIC_DATE" &&
          this.editingInstallment.due_date
            ? this.editingInstallment.due_date
            : null,
        due_event_description:
          this.editingInstallment.due_trigger_type ===
            "CUSTOM_EVENT_DESCRIPTION" &&
          this.editingInstallment.due_event_description
            ? this.editingInstallment.due_event_description.trim()
            : null,
        notes: this.editingInstallment.notes
          ? this.editingInstallment.notes.trim()
          : null,
      };

      // Basic Frontend Validation (can be enhanced)
      let validationErrors = {};
      if (!payload.amount_due || payload.amount_due <= 0)
        validationErrors.amount_due = "مبلغ القسط يجب أن يكون رقمًا موجبًا.";
      if (!payload.due_trigger_type)
        validationErrors.due_trigger_type = "يجب تحديد شرط الاستحقاق.";
      if (payload.due_trigger_type === "SPECIFIC_DATE" && !payload.due_date)
        validationErrors.due_date =
          "تاريخ الاستحقاق مطلوب عند اختيار شرط 'تاريخ محدد'.";
      if (
        payload.due_trigger_type === "CUSTOM_EVENT_DESCRIPTION" &&
        !payload.due_event_description
      )
        validationErrors.due_event_description =
          "وصف الحدث مطلوب عند اختيار شرط 'حدث مخصص'.";
      if (
        payload.due_event_description &&
        payload.due_event_description.length > 100
      )
        validationErrors.due_event_description =
          "وصف الحدث يجب ألا يتجاوز 100 حرف."; // Assuming 100 based on image

      if (Object.keys(validationErrors).length > 0) {
        this.editInstallmentError =
          "الرجاء تصحيح الأخطاء التالية:\n" +
          Object.values(validationErrors).join("\n");
        this.isSubmittingEditInstallment = false;
        return;
      }

      try {
        const response = await axios.post(
          "/api/financials/update_fee_installment.php",
          payload,
          {
            headers: { "Content-Type": "application/json" },
            withCredentials: true,
          }
        );

        if (response.data?.success) {
          this.hideEditInstallmentModal();
          this.$bvToast.toast(
            response.data.message || "تم تحديث القسط بنجاح.",
            { title: "نجاح", variant: "success", ...defaultToastOptions }
          );
          this.fetchFinancialData(); // Refresh data
        } else {
          this.editInstallmentError = this.formatApiError(response.data);
        }
      } catch (err) {
        console.error("Error submitting installment edit:", err);
        this.editInstallmentError = this.getErrorMessage(err);
        if (err.response?.status === 400 && err.response.data?.errors) {
          this.editInstallmentError =
            "فشل التحقق من صحة البيانات من الخادم: \n" +
            this.formatApiError(err.response.data);
        }
      } finally {
        this.isSubmittingEditInstallment = false;
      }
    },

    // --- Installment Status & Deletion Methods ---
    getTriggerTypeText(triggerType) {
      switch (triggerType) {
        case "SPECIFIC_DATE":
          return "تاريخ محدد";
        case "INITIAL_RULING_ISSUED":
          return "حكم ابتدائي";
        case "APPEAL_RULING_ISSUED":
          return "حكم استئناف";
        case "SUPREME_RULING_ISSUED":
          return "قرار محكمة عليا";
        case "CUSTOM_EVENT_DESCRIPTION":
          return "حدث مخصص";
        default:
          return triggerType || "غير معروف";
      }
    },
    getStatusText(statusValue) {
      const lowerStatus =
        typeof statusValue === "string" ? statusValue.toLowerCase() : "";
      switch (lowerStatus) {
        case "pending":
          return "معلق";
        case "due":
          return "مستحق";
        case "paid":
          return "مدفوع";
        case "overdue":
          return "متأخر";
        case "cancelled":
          return "ملغى";
        default:
          return statusValue || "غير معروف";
      }
    },
    getStatusVariant(statusValue) {
      const lowerStatus =
        typeof statusValue === "string" ? statusValue.toLowerCase() : "";
      switch (lowerStatus) {
        case "pending":
          return "light";
        case "due":
          return "warning";
        case "paid":
          return "success";
        case "overdue":
          return "danger";
        case "cancelled":
          return "secondary";
        default:
          return "light";
      }
    },
    async changeInstallmentStatus(installmentId, newStatus) {
      if (!installmentId || !newStatus) return;

      // Check if the status change is allowed
      if (!this.allowedManualStatuses.includes(newStatus)) {
        this.$bvToast.toast(
          `لا يمكن تغيير الحالة يدويًا إلى '${this.getStatusText(newStatus)}'.`,
          {
            title: "إجراء غير مسموح",
            variant: "danger",
            ...defaultToastOptions,
          }
        );
        return;
      }

      const installmentItem = this.financialData.installments.find(
        (inst) => inst.installment_id === installmentId
      );
      const currentStatusText = installmentItem
        ? this.getStatusText(installmentItem.status)
        : "الحالية";
      const newStatusText = this.getStatusText(newStatus);

      const confirmed = await this.$bvModal.msgBoxConfirm(
        `هل أنت متأكد من رغبتك في تغيير حالة القسط #${installmentId} من "${currentStatusText}" إلى "${newStatusText}"؟`,
        {
          title: "تأكيد تغيير الحالة",
          size: "sm",
          buttonSize: "sm",
          okVariant: "warning",
          okTitle: "نعم، قم بالتغيير",
          cancelTitle: "إلغاء",
          cancelVariant: "secondary",
          footerClass: "p-2",
          hideHeaderClose: false,
          centered: true,
        }
      );

      if (confirmed) {
        // Use $set to ensure reactivity for nested object property
        this.$set(this.isChangingStatus, installmentId, true);
        try {
          const response = await axios.post(
            "/api/financials/update_installment_status.php",
            { installment_id: installmentId, new_status: newStatus },
            {
              headers: { "Content-Type": "application/json" },
              withCredentials: true,
            }
          );

          if (response.data?.success) {
            this.$bvToast.toast(
              response.data.message ||
                `تم تغيير حالة القسط #${installmentId} إلى "${newStatusText}" بنجاح.`,
              { title: "نجاح", variant: "success", ...defaultToastOptions }
            );
            await this.fetchFinancialData(); // Refresh data after change
          } else {
            this.$bvToast.toast(this.formatApiError(response.data), {
              title: "خطأ في تغيير الحالة",
              variant: "danger",
              ...defaultToastOptions,
            });
          }
        } catch (err) {
          console.error("Error changing installment status:", err);
          this.$bvToast.toast(this.getErrorMessage(err), {
            title: "خطأ فادح",
            variant: "danger",
            ...defaultToastOptions,
          });
        } finally {
          this.$set(this.isChangingStatus, installmentId, false);
        }
      }
    },
    confirmDeleteInstallment(item) {
      if (!item || !item.installment_id) return;
      const installmentId = item.installment_id;
      const amountText = this.$options.filters.currency(item.amount_due);

      this.$bvModal
        .msgBoxConfirm(
          `هل أنت متأكد من رغبتك في حذف القسط #${installmentId} (بمبلغ ${amountText})؟ ` +
            `هذا الإجراء سيحذف القسط المجدول بشكل نهائي ولن يمكن التراجع عنه. ` +
            `ملاحظة: لن يتم الحذف إذا كان القسط مرتبطًا بدفعات مسجلة.`,
          {
            title: "تأكيد الحذف",
            size: "sm",
            buttonSize: "sm",
            okVariant: "danger",
            okTitle: "نعم، قم بالحذف",
            cancelTitle: "إلغاء",
            cancelVariant: "secondary",
            footerClass: "p-2",
            hideHeaderClose: false,
            centered: true,
          }
        )
        .then((confirmed) => {
          if (confirmed) {
            this.deleteInstallment(installmentId);
          }
        })
        .catch((err) => {
          console.error("Confirmation modal error:", err);
          this.$bvToast.toast("حدث خطأ أثناء عرض نافذة التأكيد.", {
            title: "خطأ",
            variant: "danger",
            ...defaultToastOptions,
          });
        });
    },
    async deleteInstallment(installmentId) {
      if (!installmentId) return;
      this.$set(this.isDeletingInstallment, installmentId, true);

      try {
        const response = await axios.post(
          "/api/financials/delete_fee_installment.php",
          { installment_id: installmentId },
          {
            headers: { "Content-Type": "application/json" },
            withCredentials: true,
          }
        );

        if (response.data?.success) {
          this.$bvToast.toast(
            response.data.message || `تم حذف القسط #${installmentId} بنجاح.`,
            { title: "نجاح", variant: "success", ...defaultToastOptions }
          );
          await this.fetchFinancialData(); // Refresh data
        } else {
          // Handle non-success response, including potential 409 from API (if sent with success:false)
          this.$bvToast.toast(this.formatApiError(response.data), {
            title: "خطأ في الحذف",
            variant: "danger",
            ...defaultToastOptions,
            autoHideDelay: 10000,
          });
        }
      } catch (err) {
        console.error("Error deleting installment:", err);
        // Handle error response, including 409 explicitly if needed
        if (err.response?.status === 409) {
          this.$bvToast.toast(
            err.response.data?.message ||
              "لا يمكن الحذف لوجود دفعات مرتبطة بهذا القسط.",
            {
              title: "تعذر الحذف",
              variant: "warning",
              ...defaultToastOptions,
              autoHideDelay: 10000,
            }
          );
        } else {
          this.$bvToast.toast(this.getErrorMessage(err), {
            title: "خطأ فادح",
            variant: "danger",
            ...defaultToastOptions,
          });
        }
      } finally {
        this.$set(this.isDeletingInstallment, installmentId, false);
      }
    },

    // --- Payment Methods ---
    openAddPaymentModal() {
      if (!this.financialData) {
        this.$bvToast.toast("البيانات المالية غير متاحة حاليًا.", {
          title: "خطأ",
          variant: "danger",
          ...defaultToastOptions,
        });
        return;
      }
      this.resetPaymentForm();
      this.$bvModal.show("add-payment-modal");
    },
    resetPaymentForm() {
      this.newPaymentData = { ...defaultNewPayment, case_id: this.id }; // Reset form data
      this.paymentFormErrors = null; // Clear previous errors
      const fileInput = this.$refs.paymentFileInput;
      if (fileInput) fileInput.reset(); // Reset file input
      this.isSubmittingPayment = false;
    },
    async handlePaymentSubmit() {
      this.isSubmittingPayment = true;
      this.paymentFormErrors = null; // Clear previous errors
      let payload;
      let config = { headers: {}, withCredentials: true };

      // Determine payload type: FormData if file exists, JSON otherwise
      if (this.newPaymentData.receipt_document instanceof File) {
        payload = new FormData();
        // Append all non-null/empty fields
        Object.keys(this.newPaymentData).forEach((key) => {
          if (key === "receipt_document" && this.newPaymentData[key]) {
            payload.append(key, this.newPaymentData[key]); // Append file
          } else if (
            this.newPaymentData[key] !== null &&
            this.newPaymentData[key] !== ""
          ) {
            // Ensure installment_id is sent correctly (as number or empty string for null)
            if (key === "installment_id") {
              payload.append(
                key,
                this.newPaymentData[key] === null
                  ? ""
                  : this.newPaymentData[key]
              );
            } else {
              payload.append(key, this.newPaymentData[key]);
            }
          }
        });
        config.headers["Content-Type"] = "multipart/form-data";
      } else {
        // Prepare JSON payload, explicitly handle null values
        payload = { ...this.newPaymentData };
        delete payload.receipt_document; // Remove file property if no file
        payload.payment_method = payload.payment_method || null;
        payload.installment_id =
          payload.installment_id > 0 ? payload.installment_id : null; // Send null if not selected
        payload.notes = payload.notes || null;
        config.headers["Content-Type"] = "application/json";
      }

      try {
        const response = await axios.post(
          "/api/financials/add_payment_received.php",
          payload,
          config
        );
        if (response.status === 201 && response.data?.success) {
          this.$bvToast.toast("تم تسجيل الدفعة بنجاح.", {
            title: "نجاح",
            variant: "success",
            ...defaultToastOptions,
          });
          this.$bvModal.hide("add-payment-modal");
          this.fetchFinancialData();
          this.fetchFinancialData(); // Refresh data
        } else {
          // Handle API indicating non-success (e.g., validation)
          this.paymentFormErrors = {
            general: this.formatApiError(response.data),
            validation: response.data?.errors || null,
          };
          this.$bvToast.toast("فشل تسجيل الدفعة. راجع الأخطاء.", {
            title: "خطأ",
            variant: "danger",
            ...defaultToastOptions,
          });
        }
      } catch (error) {
        // Handle network/server errors
        console.error("Error submitting payment:", error);
        const generalMessage = this.getErrorMessage(error);
        let validationErrors = null;
        // Extract validation errors if available in error response
        if (error.response?.status === 400 && error.response.data?.errors) {
          validationErrors = error.response.data.errors;
        }
        this.paymentFormErrors = {
          general: generalMessage,
          validation: validationErrors,
        };
        this.$bvToast.toast(
          "فشل تسجيل الدفعة. راجع الأخطاء في النموذج أو حاول لاحقًا.",
          {
            title: "خطأ",
            variant: "danger",
            ...defaultToastOptions,
          }
        );
      } finally {
        this.isSubmittingPayment = false;
      }
    },
    showEditPaymentModal(paymentItem) {
      if (!paymentItem || !paymentItem.payment_id) {
        console.error(
          "Invalid payment item passed to edit modal:",
          paymentItem
        );
        this.$bvToast.toast("خطأ: بيانات الدفعة غير صالحة للتعديل.", {
          title: "خطأ داخلي",
          variant: "danger",
          ...defaultToastOptions,
        });
        return;
      }
      // Deep clone the payment item
      this.editingPayment = JSON.parse(JSON.stringify(paymentItem));

      // Format date for datepicker (adjust for timezone)
      if (this.editingPayment.payment_date) {
        try {
          const date = new Date(this.editingPayment.payment_date);
          if (!isNaN(date.getTime())) {
            const timezoneOffset = date.getTimezoneOffset() * 60000;
            const adjustedDate = new Date(date.getTime() - timezoneOffset);
            this.editingPayment.payment_date = adjustedDate
              .toISOString()
              .split("T")[0];
          } else {
            this.editingPayment.payment_date = null; // Clear invalid date
          }
        } catch (e) {
          console.error("Error parsing payment date for edit:", e);
          this.editingPayment.payment_date = null;
        }
      }
      // Ensure installment_id is null if 0 or undefined
      this.editingPayment.installment_id =
        this.editingPayment.installment_id || null;

      // Store current receipt details for display/deletion logic
      this.currentReceiptDetails = this.editingPayment.receipt_document_id
        ? { id: this.editingPayment.receipt_document_id }
        : null;

      // Reset edit modal state
      this.editPaymentError = null;
      this.isSubmittingEditPayment = false;
      this.editPaymentReceiptFile = null; // Clear any previous file selection
      this.removeCurrentReceipt = false; // Uncheck deletion box
      if (this.$refs.editPaymentFileInput) {
        // Reset file input visually
        this.$refs.editPaymentFileInput.reset();
      }

      this.$bvModal.show("edit-payment-modal");
    },
    hideEditPaymentModal() {
      this.$bvModal.hide("edit-payment-modal");
    },
    resetEditPaymentForm() {
      this.editingPayment = defaultEditingPayment; // Reset to null
      this.editPaymentError = null;
      this.isSubmittingEditPayment = false;
      this.editPaymentReceiptFile = null;
      this.removeCurrentReceipt = false;
      this.currentReceiptDetails = null;
      if (this.$refs.editPaymentFileInput) {
        this.$refs.editPaymentFileInput.reset();
      }
    },
    async submitEditPayment() {
      if (!this.editingPayment || !this.editingPayment.payment_id) {
        this.editPaymentError = "خطأ: لا يمكن تحديد الدفعة المراد تعديلها.";
        return;
      }

      this.isSubmittingEditPayment = true;
      this.editPaymentError = null;

      let payload;
      let config = { headers: {}, withCredentials: true };
      const hasNewFile = this.editPaymentReceiptFile instanceof File;

      // Determine payload type: FormData if new file, JSON otherwise
      if (hasNewFile) {
        payload = new FormData();
        config.headers["Content-Type"] = "multipart/form-data";

        // Append required fields
        payload.append("payment_id", this.editingPayment.payment_id);
        payload.append("payment_date", this.editingPayment.payment_date || "");
        payload.append(
          "amount_received",
          this.editingPayment.amount_received || 0
        );

        // Append optional fields if they have values
        if (this.editingPayment.payment_method) {
          payload.append("payment_method", this.editingPayment.payment_method);
        }
        // Ensure installment_id is sent correctly (empty string for null)
        payload.append(
          "installment_id",
          this.editingPayment.installment_id === null
            ? ""
            : this.editingPayment.installment_id
        );
        if (this.editingPayment.notes) {
          payload.append("notes", this.editingPayment.notes);
        }
        // Append the new file
        payload.append("receipt_document", this.editPaymentReceiptFile);
        // Note: remove_receipt is implicitly false when a new file is uploaded
      } else {
        // Prepare JSON payload
        config.headers["Content-Type"] = "application/json";
        payload = {
          payment_id: this.editingPayment.payment_id,
          payment_date: this.editingPayment.payment_date || null,
          amount_received: this.editingPayment.amount_received || 0,
          payment_method: this.editingPayment.payment_method || null,
          installment_id: this.editingPayment.installment_id || null,
          notes: this.editingPayment.notes || null,
        };
        // Add remove_receipt flag only if the checkbox is checked and there was a current receipt
        if (this.removeCurrentReceipt && this.currentReceiptDetails) {
          payload.remove_receipt = true;
        }
      }

      try {
        const response = await axios.post(
          "/api/financials/update_payment_received.php",
          payload,
          config
        );

        if (response.data?.success) {
          this.hideEditPaymentModal();
          this.$bvToast.toast(
            response.data.message || "تم تحديث الدفعة بنجاح.",
            { title: "نجاح", variant: "success", ...defaultToastOptions }
          );
          await this.fetchFinancialData(); // Refresh data
        } else {
          this.editPaymentError = this.formatApiError(response.data);
          this.$bvToast.toast("فشل تحديث الدفعة.", {
            title: "خطأ",
            variant: "danger",
            ...defaultToastOptions,
          });
        }
      } catch (err) {
        console.error("Error updating payment:", err);
        this.editPaymentError = this.getErrorMessage(err);
        this.$bvToast.toast(
          "فشل تحديث الدفعة. يرجى مراجعة الأخطاء أو المحاولة لاحقاً.",
          {
            title: "خطأ",
            variant: "danger",
            ...defaultToastOptions,
          }
        );
      } finally {
        this.isSubmittingEditPayment = false;
      }
    },
    async confirmDeletePayment(paymentItem) {
      if (!paymentItem || !paymentItem.payment_id) return;
      const paymentId = paymentItem.payment_id;
      const amountText = this.$options.filters.currency(
        paymentItem.amount_received
      );
      const dateText = this.formatDate(paymentItem.payment_date);

      const confirmed = await this.$bvModal.msgBoxConfirm(
        `هل أنت متأكد من رغبتك في حذف الدفعة المستلمة #${paymentId} (بمبلغ ${amountText} بتاريخ ${dateText})؟ ` +
          `هذا الإجراء سيحذف الدفعة والإيصال المرفق بها (إن وجد) بشكل نهائي ولن يمكن التراجع عنه.`,
        {
          title: "تأكيد حذف الدفعة",
          size: "md",
          buttonSize: "sm",
          okVariant: "danger",
          okTitle: "نعم، قم بالحذف",
          cancelTitle: "إلغاء",
          cancelVariant: "secondary",
          footerClass: "p-2",
          hideHeaderClose: false,
          centered: true,
        }
      );

      if (confirmed) {
        this.deletePayment(paymentId);
      }
    },
    async deletePayment(paymentId) {
      if (!paymentId) return;
      this.$set(this.isDeletingPayment, paymentId, true); // Use $set for reactivity

      try {
        const response = await axios.post(
          "/api/financials/delete_payment_received.php",
          { payment_id: paymentId },
          {
            headers: { "Content-Type": "application/json" },
            withCredentials: true,
          }
        );

        if (response.data?.success) {
          this.$bvToast.toast(
            response.data.message || `تم حذف الدفعة #${paymentId} بنجاح.`,
            { title: "نجاح", variant: "success", ...defaultToastOptions }
          );
          await this.fetchFinancialData(); // Refresh data
        } else {
          // Handle API error (success: false)
          console.error(
            "API Error deleting payment (success:false):",
            response.data
          );
          this.$bvToast.toast(this.formatApiError(response.data), {
            title: "خطأ في الحذف",
            variant: "danger",
            ...defaultToastOptions,
          });
        }
      } catch (err) {
        // Handle network/server error
        console.error("Error deleting payment:", err);
        this.$bvToast.toast(this.getErrorMessage(err), {
          title: "خطأ فادح",
          variant: "danger",
          ...defaultToastOptions,
        });
      } finally {
        this.$set(this.isDeletingPayment, paymentId, false); // Use $set for reactivity
      }
    },

    // --- Contract Deletion ---
    confirmDeleteContract() {
      const documentId = this.financialData?.contract_document?.document_id;
      const fileName =
        this.financialData?.contract_document?.file_name_original;
      if (!documentId) return; // Should not happen if button is shown

      this.$bvModal
        .msgBoxConfirm(
          `هل أنت متأكد من رغبتك في حذف ملف العقد "${
            fileName || "المحدد"
          }"؟ هذا الإجراء لا يمكن التراجع عنه.`,
          {
            title: "تأكيد الحذف",
            size: "sm",
            buttonSize: "sm",
            okVariant: "danger",
            okTitle: "نعم، قم بالحذف",
            cancelTitle: "إلغاء",
            cancelVariant: "secondary",
            footerClass: "p-2",
            hideHeaderClose: false,
            centered: true,
          }
        )
        .then((confirmed) => {
          if (confirmed) this.deleteContractDocument(documentId);
        })
        .catch((err) => {
          console.error("Modal Error:", err);
          this.$bvToast.toast("حدث خطأ أثناء عرض نافذة التأكيد.", {
            title: "خطأ",
            variant: "danger",
            ...defaultToastOptions,
          });
        });
    },
    async deleteContractDocument(docIdToDelete) {
      if (!docIdToDelete) {
        this.$bvToast.toast("خطأ: لا يمكن تحديد معرف المستند للحذف.", {
          title: "خطأ داخلي",
          variant: "danger",
          ...defaultToastOptions,
        });
        return;
      }
      // Add loading state if needed
      try {
        const response = await axios.post(
          "/api/financials/delete_contract_document.php",
          { document_id: docIdToDelete },
          {
            headers: { "Content-Type": "application/json" },
            withCredentials: true,
          }
        );
        if (response.data?.success) {
          this.$bvToast.toast(
            response.data.message || "تم حذف ملف العقد بنجاح.",
            { title: "نجاح", variant: "success", ...defaultToastOptions }
          );
          this.fetchFinancialData(); // Refresh to update the contract section
        } else {
          const message = response.data?.message || "فشل حذف ملف العقد.";
          console.error("API Error deleting contract:", response.data);
          this.$bvToast.toast(message, {
            title: "خطأ",
            variant: "danger",
            ...defaultToastOptions,
          });
        }
      } catch (error) {
        console.error("Error deleting contract document:", error);
        const errorMessage = this.getErrorMessage(error);
        this.$bvToast.toast(errorMessage, {
          title: "خطأ فادح",
          variant: "danger",
          ...defaultToastOptions,
        });
      }
    },

    // --- Agreement Deletion Methods ---
    async confirmDeleteAgreement() {
      if (!this.financialData?.agreement?.agreement_id) return;
      const agreementId = this.financialData.agreement.agreement_id;
      const agreementTypeText = this.getAgreementTypeText(
        this.financialData.agreement.agreement_type
      );

      const confirmed = await this.$bvModal.msgBoxConfirm(
        `هل أنت متأكد من رغبتك في حذف اتفاقية الأتعاب (#${agreementId} - ${agreementTypeText}) بالكامل؟\n\n` +
          `تحذير هام: هذا الإجراء سيحذف الاتفاقية والعقد المرفق بها (إن وجد) بشكل نهائي ولن يمكن التراجع عنه.\n\n` +
          `ملاحظة هامة: لا يمكن حذف الاتفاقية إذا كانت مرتبطة بأي أقساط مجدولة. يجب حذف الأقساط المرتبطة أولاً.`, // Updated constraint based on SSOT
        {
          title: "تأكيد حذف اتفاقية الأتعاب",
          size: "lg", // Larger size for more text
          buttonSize: "sm",
          okVariant: "danger",
          okTitle: "نعم، قم بالحذف",
          cancelTitle: "إلغاء",
          cancelVariant: "secondary",
          footerClass: "p-2",
          hideHeaderClose: false,
          centered: true,
          bodyClass: "text-danger font-weight-bold text-right", // Make text prominent and right-aligned
          msgBoxBodyClass: "text-right", // Ensure body content alignment
        }
      );

      if (confirmed) {
        this.deleteAgreement(agreementId);
      }
    },
    async deleteAgreement(agreementId) {
      if (!agreementId) return;
      this.isDeletingAgreement = true;

      try {
        const response = await axios.post(
          "/api/financials/delete_fee_agreement.php",
          { agreement_id: agreementId },
          {
            headers: { "Content-Type": "application/json" },
            withCredentials: true,
          }
        );

        if (response.data?.success) {
          // Success (status 200 typically)
          this.$bvToast.toast(
            response.data.message ||
              `تم حذف اتفاقية الأتعاب #${agreementId} بنجاح.`,
            { title: "نجاح", variant: "success", ...defaultToastOptions }
          );
          // Reset financial data completely after successful deletion
          this.resetFinancialDataOnError(); // Use the existing reset function
          // No need to call fetchFinancialData() as everything related is gone.
        } else {
          // Handle API indicating non-success (e.g., success: false, status 409, 400, 500 etc.)
          console.error(
            "API Error deleting agreement (success:false):",
            response.data
          );
          this.$bvToast.toast(
            this.formatApiError(response.data), // Use the message from API
            {
              title: "خطأ في الحذف",
              variant: "danger",
              ...defaultToastOptions,
              autoHideDelay: 10000,
            } // Longer delay
          );
        }
      } catch (err) {
        // Handle network/server errors, including 409 Conflict via getErrorMessage
        console.error("Error deleting agreement:", err);
        this.$bvToast.toast(
          this.getErrorMessage(err), // getErrorMessage handles 409 from error response
          {
            title: "خطأ فادح",
            variant: "danger",
            ...defaultToastOptions,
            autoHideDelay: 10000,
          } // Longer delay
        );
      } finally {
        this.isDeletingAgreement = false;
      }
    },

    // --- Utility Methods ---
    formatDate(dateString) {
      if (!dateString) return "-";
      const date = new Date(dateString);
      // Check if date is valid
      if (isNaN(date.getTime())) return dateString; // Return original string if invalid
      try {
        // Adjust for timezone offset before formatting to display the intended date
        const timezoneOffset = date.getTimezoneOffset() * 60000;
        const adjustedDate = new Date(date.getTime() + timezoneOffset);

        const options = {
          year: "numeric",
          month: "long",
          day: "numeric",
          timeZone: "UTC",
        }; // Use UTC after adjustment
        return adjustedDate.toLocaleDateString("ar-SA", options);
      } catch (e) {
        console.error("Error formatting date:", e);
        return dateString; // Fallback to original string on error
      }
    },
    truncateText(text, length) {
      if (!text) return "-";
      if (text.length <= length) return text;
      return text.substring(0, length) + "...";
    },
    formatApiError(data) {
      // Start with the main message or a default
      let message = data?.message || "حدث خطأ غير متوقع.";
      // Append details from 'errors' if present
      if (data?.errors) {
        let errorDetails = "";
        if (Array.isArray(data.errors)) {
          errorDetails = data.errors.join("\n");
        } else if (typeof data.errors === "object" && data.errors !== null) {
          // Format object errors (e.g., { field: 'message' })
          errorDetails = Object.entries(data.errors)
            .map(([key, value]) => `${key}: ${value}`)
            .join("\n");
        } else if (typeof data.errors === "string") {
          errorDetails = data.errors;
        }

        // Prepend a validation message if appropriate
        if (errorDetails) {
          if (
            message.toLowerCase().includes("validation error") ||
            message.toLowerCase().includes("validation failed") ||
            message.toLowerCase().includes("فشل التحقق") // Arabic check
          ) {
            message = `فشل التحقق من صحة البيانات:\n${errorDetails}`;
          } else if (!message.endsWith("\n")) {
            // Add details nicely if not a primary validation message
            message += `\nالتفاصيل:\n${errorDetails}`;
          } else {
            message += `التفاصيل:\n${errorDetails}`;
          }
        }
      }
      return message.trim(); // Trim whitespace
    },
    // Helper for checking validation errors in payment form specifically
    hasValidationError(fieldName) {
      if (this.paymentFormErrors?.validation) {
        // Check if errors is an object (key: message)
        if (
          typeof this.paymentFormErrors.validation === "object" &&
          this.paymentFormErrors.validation !== null &&
          !Array.isArray(this.paymentFormErrors.validation)
        ) {
          return !!this.paymentFormErrors.validation[fieldName];
        }
        // Check if errors is an array of strings
        else if (Array.isArray(this.paymentFormErrors.validation)) {
          return this.paymentFormErrors.validation.some(
            (e) =>
              typeof e === "string" &&
              e.toLowerCase().includes(fieldName.toLowerCase())
          );
        }
      }
      return false;
    },
  },
  filters: {
    currency(value) {
      if (typeof value !== "number") {
        const numValue = parseFloat(value);
        if (isNaN(numValue)) return value; // Return original value if not parsable
        value = numValue;
      }
      // Use Intl.NumberFormat for robust currency formatting
      const formatter = new Intl.NumberFormat("ar-SA", {
        style: "currency",
        currency: "SAR", // Saudi Riyal
        minimumFractionDigits: 2,
        maximumFractionDigits: 2,
      });
      return formatter.format(value);
    },
  },
};
</script>
<style scoped>
.card-header button {
  width: 100%;
  text-align: right;
}
/* Updated rule to target h5 inside card-header directly */
.card-header > h5 {
  text-align: right;
  margin-bottom: 0; /* Remove default margin if needed */
}
.alert {
  direction: rtl;
  text-align: right;
  white-space: pre-wrap; /* Preserve line breaks in error messages */
}
/* Ensure labels in modals are right-aligned */
#add-agreement-modal .form-group label,
#edit-agreement-modal .form-group label,
#link-contract-modal .form-group label,
#add-installment-modal .form-group label,
#edit-installment-modal .form-group label,
#add-payment-modal .form-group label,
#edit-payment-modal .form-group label {
  text-align: right;
  display: block; /* Ensure label takes full width */
}
.form-group {
  margin-bottom: 1rem; /* Standard spacing */
}
.table {
  text-align: right; /* Default text alignment for table cells */
  vertical-align: middle; /* Align cell content vertically */
}
.table th.text-center, /* Center align specific headers */
.table td.text-center {
  /* Center align specific cells */
  text-align: center !important;
}
.table thead th {
  vertical-align: middle; /* Ensure header text is aligned */
}
/* Remove default Bootstrap sort icons */
.table
  thead
  th:not(.b-table-sort-icon-left):not(.b-table-sort-icon-right)::before,
.table
  thead
  th:not(.b-table-sort-icon-left):not(.b-table-sort-icon-right)::after {
  content: none !important;
}
.table thead.table-light th {
  background-color: #f8f9fa; /* Standard light header background */
}
/* Style lists within modal alerts */
#add-agreement-modal .alert ul,
#edit-agreement-modal .alert ul,
#link-contract-modal .alert ul,
#add-installment-modal .alert ul,
#edit-installment-modal .alert ul,
#add-payment-modal .alert ul,
#edit-payment-modal .alert ul {
  margin-bottom: 0;
  padding-right: 1.5rem; /* Indent list items */
}
/* Style headings and paragraphs within modal alerts */
#add-agreement-modal .alert h5,
#edit-agreement-modal .alert h5,
#link-contract-modal .alert h5,
#add-installment-modal .alert h5,
#edit-installment-modal .alert h5,
#add-payment-modal .alert h5,
#edit-payment-modal .alert h5 {
  margin-bottom: 0.5rem;
}
#add-agreement-modal .alert p,
#edit-agreement-modal .alert p,
#link-contract-modal .alert p,
#add-installment-modal .alert p,
#edit-installment-modal .alert p,
#add-payment-modal .alert p,
#edit-payment-modal .alert p {
  margin-bottom: 0.5rem;
  white-space: pre-wrap; /* Preserve line breaks */
}
#add-agreement-modal .alert p:last-child,
#edit-agreement-modal .alert p:last-child,
#link-contract-modal .alert p:last-child,
#add-installment-modal .alert p:last-child,
#edit-installment-modal .alert p:last-child,
#add-payment-modal .alert p:last-child,
#edit-payment-modal .alert p:last-child {
  margin-bottom: 0; /* Remove margin from last paragraph */
}
/* Ensure status dropdown button content is centered */
.status-dropdown .btn {
  width: 100%; /* Make button fill cell width */
  display: flex !important; /* Use flex for alignment */
  justify-content: center;
  align-items: center;
}
.status-dropdown .dropdown-menu {
  min-width: auto; /* Prevent dropdown from being too wide */
}
/* Define column widths for better table layout */
.table .status-col {
  width: 130px; /* Fixed width for status column */
}
.table .actions-col {
  width: 120px; /* Fixed width for actions column */
}
/* Style checkbox in edit payment modal */
#edit-payment-modal .form-check {
  padding-right: 1.25rem; /* Align with Bootstrap's RTL styling */
}
#edit-payment-modal .form-check .form-check-input {
  float: right;
  margin-left: 0;
  margin-right: -1.25rem; /* Position checkbox correctly */
}
#edit-payment-modal .form-check .form-check-label {
  padding-right: 0.5em; /* Space between checkbox and label */
}
/* Add some style for the summary cards */
.card-text h4 {
  font-weight: bold;
  margin-top: 0.25rem;
}
.card-text h5 {
  color: #6c757d; /* Muted color for the title */
}
.card-text small {
  display: block;
  margin-top: 0.25rem;
  font-style: italic;
}
/* Make collapse headers look more clickable */
.card-header button[aria-expanded="false"] .fa-chevron-down {
  transform: rotate(0deg);
  transition: transform 0.3s ease;
}
.card-header button[aria-expanded="true"] .fa-chevron-down {
  transform: rotate(180deg);
  transition: transform 0.3s ease;
}
.card-header button {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.float-left {
  /* Ensure icon + text stays left even in RTL */
  float: left !important;
  margin-right: auto; /* Push chevron to the right */
}
.fa-calculator,
.fa-file-contract,
.fa-tasks,
.fa-hand-holding-usd {
  margin-left: 0.5rem; /* Add space after icon in RTL context */
}
</style>
