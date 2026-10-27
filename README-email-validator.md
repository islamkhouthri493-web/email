# أداة التحقق من صحة البريد الإلكتروني (Email Validation Tool)

أداة متكاملة مبنية بـ **TypeScript / Next.js (App Router)** لتُستخدم كخدمة ويب (API + واجهة
مستخدم) إضافةً إلى أداة سطر أوامر (CLI) مستقلة.

## البنية (Modular Architecture)

```
src/lib/email-validator/
  ├── types.ts            # الأنواع المشتركة (Result, Options, ReasonCode...)
  ├── core.ts             # الدالة الرئيسية validateEmail() + كل خطوات الفحص
  ├── bulk.ts             # التحقق الجماعي + استخراج العناوين من CSV/نص
  ├── common-domains.ts   # اقتراح تصحيح الأخطاء الإملائية (Levenshtein)
  └── index.ts            # نقطة تصدير موحّدة

src/app/api/validate-email/
  ├── route.ts                 # POST  فحص فردي
  ├── bulk/route.ts            # POST  فحص جماعي (JSON أو ملف CSV)
  ├── history/route.ts         # GET   قائمة عمليات التحقق السابقة
  └── history/[id]/route.ts    # GET   تفاصيل عملية تحقق محددة

src/components/email-validator/  # واجهة المستخدم (تبويبات: فردي/جماعي/سجل)

scripts/validate-email-cli.mjs   # أداة سطر أوامر مستقلة (لا تحتاج بناء المشروع)
```

## الدالة الرئيسية

```ts
import { validateEmail } from "@/lib/email-validator";

const result = await validateEmail("user@gmial.com", {
  checkDomain: true, // DNS Lookup
  checkMx: true, // MX Records
  checkSmtp: false, // تجريبي، معطّل افتراضيًا
  rejectDisposable: true, // رفض الإيميلات المؤقتة
});

/*
{
  email: "user@gmial.com",
  normalizedEmail: "user@gmial.com",
  valid: true,
  reasonCode: null,
  reasonMessage: "البريد الإلكتروني صالح وفق جميع الفحوصات المُفعّلة.",
  checks: { format: true, length: true, disposable: false, mxRecord: true, domainExists: true, smtpMailbox: null },
  suggestion: "gmail.com", // اقتراح تصحيح احتمالي
  durationMs: 42
}
*/
```

### أكواد الفشل الممكنة (`reasonCode`)

`empty`, `too_long`, `contains_whitespace`, `multiple_at`, `missing_at`,
`missing_local_part`, `missing_domain_part`, `local_part_too_long`,
`domain_missing_dot`, `invalid_format`, `disposable_email`, `domain_not_found`,
`no_mx_record`, `smtp_mailbox_not_found`, `smtp_check_inconclusive`.

## التحقق الجماعي (Bulk)

```ts
import { validateEmailsBulk, extractEmailsFromText } from "@/lib/email-validator";

const emails = extractEmailsFromText(csvFileContent); // يدعم عمود "email" أو قائمة بسيطة
const { results, summary } = await validateEmailsBulk(emails, { checkSmtp: false });
```

## واجهات الـ API

- `POST /api/validate-email` — body: `{ "email": "...", "checkMx": true, "checkDomain": true, "checkSmtp": false, "rejectDisposable": true }`
- `POST /api/validate-email/bulk` — body JSON: `{ "emails": ["a@x.com", ...] }` أو `{ "text": "..." }`، أو `multipart/form-data` بحقل `file` (CSV).
- `GET /api/validate-email/history` — آخر 50 عملية تحقق.
- `GET /api/validate-email/history/:id` — تفاصيل عملية تحقق معينة.

كل عملية تحقق (فردية أو جماعية) تُخزَّن في PostgreSQL عبر Drizzle
(`email_validation_batches` و`email_validation_results`) لتظهر في تبويب "السجل" بالواجهة.

## أداة سطر الأوامر (CLI)

لا تحتاج بناء المشروع، تعمل مباشرة عبر Node:

```bash
node scripts/validate-email-cli.mjs user@example.com another@gmail.com
node scripts/validate-email-cli.mjs --file emails.csv --json
cat emails.txt | node scripts/validate-email-cli.mjs --smtp
```

الخيارات: `--file <path>`, `--smtp`, `--no-mx`, `--no-domain`, `--allow-disposable`, `--json`.

## ملاحظات تقنية مهمة

1. **الفرق بين "صيغة صحيحة" و"بريد موجود فعليًا"**: صحة الصيغة (Syntax) تعني فقط أن النص
   مطابق لشكل بريد إلكتروني مقبول، ولا تضمن وجود الصندوق البريدي. فحص MX/الدومين يؤكد أن
   الدومين *قادر نظريًا* على استقبال البريد. التأكد الكامل من وجود الصندوق نفسه ممكن فقط
   تقريبيًا عبر SMTP (اختياري) لأن أغلب المزوّدين (Gmail, Outlook...) يخفون هذه المعلومة
   عمدًا لمنع "حصاد" العناوين، كما أن كثيرًا من الشبكات (ومنها البيئات السحابية) تحجب منفذ
   SMTP الصادر 25، لذا تُعامل نتيجته كمؤشر غير حاسم وليست حقيقة مطلقة.
2. تُستخدم مكتبتا **validator** (فحص الصيغة) و**disposable-email-domains** (قائمة تضم أكثر
   من 120,000 دومين بريد مؤقت) كمكتبات موثوقة بدل إعادة اختراع العجلة.
3. الكود Modular بالكامل: يمكن إضافة فحص جديد (مثل API خارجي لفحص السمعة) بإضافة ملف جديد في
   `src/lib/email-validator/` وتصديره من `index.ts` دون تعديل بقية الكود.
