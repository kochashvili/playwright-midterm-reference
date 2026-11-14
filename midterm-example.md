# პროგრამული უზრუნველყოფის ტესტირება შუალედური გამოცდა · სანიმუშო ვარიანტი

> ეს არის **მაგალითი**: ასე გამოიყურება გამოცდის ბილეთი. ამოხსნა — ფოლდერი `midterm-example/` (ტესტ-ქეისები `test-cases-example.xlsx`, კოდი `tests/`). გამოცდაზე სხვა საიტები და სხვა დავალებები იქნება.

ნაწილი I: ტესტ-ქეისები Excel-ში (5 ქულა) · ნაწილი II: ტესტირების ავტომატიზაცია (20 ქულა)

---

## 1. შექმენი პროექტი და დააყენე ყველა dependency

დახმარება: `SETUP-UV.md` და ლექციის მაგალითები შეგიძლიათ ნახოთ softwere-testing-lectures.zip-ის ამოარქივების შემდგომ `playwright-lectures-code-repository` პროექტში

## uv-ის ჩართვა აუდიტორიის კომპიუტერზე

როცა PyCharm-ის შექმნით პროექტს გახსენით მაგ პროექტში PowerShell ტერმინალი და გაწერეთ:

```powershell
$env:Path += ";C:\Users\student\AppData\Roaming\Python\Scripts"
uv --version
```

---

## 2. ნაწილი I · ხელით ტესტირება, ავტომატიზაციის გარეშე (5 ქულა)

აღწერეთ **ორი** ტესტ-ქეისი Excel-ში და შეასრულე ისინი **ხელით**, ბრაუზერში. ამ საიტისთვის კოდი არ არის საჭირო.

- საიტი: **Demoblaze (ონლაინ მაღაზია)** — [https://www.demoblaze.com/](https://www.demoblaze.com/)

| #     | ტიპი     | სცენარი                                                                    | მონაცემები          |
|:----- |:-------- |:-------------------------------------------------------------------------- |:------------------- |
| TC-01 | Positive | ტელეფონის დამატება კალათაში: შეამოწმე alert-ი, კალათის სტრიქონი და `Total` | `Samsung galaxy s6` |
| TC-02 | Negative | `Place Order` ცარიელი ფორმით → `Purchase`; შეკვეთა არ უნდა შედგეს         | ცარიელი ველები      |

- შაბლონი: `test-case-template.xlsx`
- შეავსე ყველა 9 ველი ქართულად: ID, Title, Type, Preconditions, Steps, Test data, Expected result, Actual result, Status
- Steps: 5–8 ნაბიჯი, ყოველი ნაბიჯი — ერთი მოქმედება
- Expected result უნდა იყოს ზუსტი, მაგალითად შეტყობინების ტექსტი სიტყვასიტყვით
- Actual result და Status შეავსე მხოლოდ ტესტის **შემოწმების შემდეგ** (`Pass` / `Fail`)
- ფაილის სახელი: `test-cases-სახელი-გვარი.xlsx`

---

## 3. ნაწილი II · ავტომატიზაცია (20 ქულა)

### დავალება 1 · Checkbox-ები

- საიტი: [https://practice.expandtesting.com/checkboxes](https://practice.expandtesting.com/checkboxes)
- შემოწმება:
  - თავიდან `#checkbox1` არ არის მონიშნული, `#checkbox2` მონიშნულია
  - მონიშნე `#checkbox1` და მოხსენი `#checkbox2` → შეამოწმე ორივე
- მინიშნება: `check()`, `uncheck()`, `to_be_checked()`, `not_to_be_checked()`

### დავალება 2 · კლავიშები

- საიტი: [https://practice.expandtesting.com/key-presses](https://practice.expandtesting.com/key-presses)
- ველში `#target` დააჭირე კლავიშს → `#result`-ში ტექსტი:

| კლავიში  | `#result`             |
|:-------- |:--------------------- |
| `Escape` | `You entered: ESCAPE` |
| `a`      | `You entered: A`      |

- მინიშნება: `locator.press("Escape")` (ლექცია 5)

### დავალება 3 · კალათის მთვლელი parametrize-ით

- საიტი: [https://www.saucedemo.com/](https://www.saucedemo.com/) · მონაცემები: standard_user / secret_sauce
- შესვლა გააკეთე fixture-ში. დაწერე **ერთი** ტესტ-ფუნქცია `@pytest.mark.parametrize`-ით: დაამატე პროდუქტები კალათაში → `[data-test="shopping-cart-badge"]` უდრის პროდუქტების რაოდენობას

| პროდუქტები                                                          | კალათის ნიშანი |
|:------------------------------------------------------------------- |:-------------- |
| `Sauce Labs Backpack`                                               | `1`            |
| `Sauce Labs Backpack`, `Sauce Labs Bike Light`                      | `2`            |
| `Sauce Labs Backpack`, `Sauce Labs Bike Light`, `Sauce Labs Onesie` | `3`            |

- მინიშნება: პროდუქტის ბარათი → `filter(has_text=...)`, შიგნით ღილაკი `Add to cart`
- შედეგი: `3 passed` ამ ფუნქციისთვის

### დავალება 4 · ახალი ჩანართი და ახალი ფანჯარა (Optional)

- საიტი: [https://demoqa.com/browser-windows](https://demoqa.com/browser-windows)
- ორი შემთხვევა (`New Tab`, `New Window`), ორივე ერთი ტესტ-ფუნქციით:
  - დააჭირე ღილაკს → იხსნება ახალი გვერდი
  - ახალი გვერდის URL მთავრდება `/sample`-ით, `#sampleHeading` არის `This is a sample page`
  - დახურე ახალი გვერდი → პირველი გვერდის URL ისევ `https://demoqa.com/browser-windows`
- მინიშნება: `with page.expect_popup() as info:` (ლექცია 4)

### დავალება 5 · კალათა Page Object-ებით (Optional)

- საიტი: [https://www.saucedemo.com/](https://www.saucedemo.com/) · მონაცემები: standard_user / secret_sauce
- შექმენი კლასები `tests/pages/`-ში: `LoginPage`, `InventoryPage`, `CartPage`. ტესტში პირდაპირ `page.locator(...)` არ უნდა იყოს.
- ნაბიჯები და შემოწმება:
  1. შედი სისტემაში, დაამატე `Sauce Labs Backpack`, `Sauce Labs Bike Light`, `Sauce Labs Onesie` → ნიშანი `3`
  2. გახსენი კალათა → 3 პროდუქტი
  3. წაშალე `Sauce Labs Bike Light` (`Remove`) → დარჩა `Sauce Labs Backpack` და `Sauce Labs Onesie`, ნიშანი `2`
  4. `Continue Shopping` → URL `https://www.saucedemo.com/inventory.html`, სათაური `Products`
  5. მენიუ (`#react-burger-menu-btn`) → `Logout` → URL `https://www.saucedemo.com/`, ღილაკი `Login` ჩანს

---

## 4. გაუშვი ტესტები და შექმენი რეპორტი

✅ შედეგი: ყველა ტესტი `passed`. ფაილი `report.html`

აღწერეთ ცალკე ფაილში commands.txt რა ბრძანებები გამოიყენეთ

---

## 5. გამომიგზავნე ZIP არქივად

არქივის სახელი: `სახელი-გვარი-midterm.zip`

არქივში უნდა იყოს:

- `pyproject.toml`
- `uv.lock`
- `tests/` ფოლდერი (`test_task1_...py` – `test_task5_...py`, `pages/`, `conftest.py`)
- `report.html`
- `commands.txt`
- `test-cases-სახელი-გვარი.xlsx` (ნაწილი I)

⚠️ `.venv` ფოლდერი არ ჩადო არქივში. ის დიდია და მხოლოდ dependencies-ს შეიცავს, რაც საჭირო არ არის.
